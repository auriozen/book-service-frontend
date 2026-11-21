<script setup>
import { ref, computed, onMounted } from "vue";

const API_URL = "http://localhost:8080/books";

const books = ref([]);
const search = ref("");
const loading = ref(false);
const error = ref("");
const showForm = ref(false);
const editingId = ref(null);

const form = ref({
  title: "",
  author: "",
  price: "",
  imageUrl: "",
});

const filteredBooks = computed(() =>
  books.value.filter((book) =>
    `${book.title} ${book.author}`
      .toLowerCase()
      .includes(search.value.toLowerCase()),
  ),
);

async function loadBooks() {
  loading.value = true;
  error.value = "";

  try {
    const response = await fetch(API_URL);

    if (!response.ok) {
      throw new Error("Unable to load books.");
    }

    books.value = await response.json();
  } catch (err) {
    error.value = err.message;
  } finally {
    loading.value = false;
  }
}

function openAddForm() {
  editingId.value = null;
  form.value = {
    title: "",
    author: "",
    price: "",
    imageUrl: "",
  };
  showForm.value = true;
}

function editBook(book) {
  editingId.value = book.id;
  form.value = {
    title: book.title,
    author: book.author,
    price: book.price,
    imageUrl: book.imageUrl || "",
  };
  showForm.value = true;
}

async function saveBook() {
  error.value = "";

  const editing = editingId.value !== null;
  const url = editing ? `${API_URL}/${editingId.value}` : API_URL;

  try {
    const response = await fetch(url, {
      method: editing ? "PUT" : "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        ...form.value,
        price: Number(form.value.price),
      }),
    });

    if (!response.ok) {
      throw new Error("Unable to save book.");
    }

    showForm.value = false;
    await loadBooks();
  } catch (err) {
    error.value = err.message;
  }
}

async function deleteBook(book) {
  if (!confirm(`Delete "${book.title}"?`)) return;

  try {
    const response = await fetch(`${API_URL}/${book.id}`, {
      method: "DELETE",
    });

    if (!response.ok) {
      throw new Error("Unable to delete book.");
    }

    await loadBooks();
  } catch (err) {
    error.value = err.message;
  }
}

onMounted(loadBooks);
</script>

<template>
  <div class="app-layout">
    <aside class="sidebar">
      <div class="brand">
        <span class="brand-icon">B.</span>
        <div>
          <h2>Books-R-Us</h2>
          <small>Library Management</small>
        </div>
      </div>

      <div class="nav-heading">WORKSPACE</div>
      <nav>
        <a class="nav-active">▦ &nbsp; Book Collection</a>
      </nav>

      <div class="sidebar-bottom">
        <p>Books-R-Us</p>
        <small>Manage your collection.</small>
      </div>
    </aside>

    <main class="main-content">
      <header class="topbar">
        <span>Library / Collection</span>
        <div class="avatar">BR</div>
      </header>

      <section class="content">
        <div class="welcome">
          <div>
            <p class="eyebrow">YOUR LIBRARY</p>
            <h1>Book Management</h1>
            <p class="subtitle">
              Organize, discover, and manage your book collection.
            </p>
          </div>
          <button class="primary-btn" @click="openAddForm">
            + Add New Book
          </button>
        </div>

        <section class="stats">
          <div class="stat-card">
            <span>Total Books</span>
            <strong>{{ books.length }}</strong>
            <small>Books in your collection</small>
          </div>
          <div class="stat-card">
            <span>Unique Authors</span>
            <strong>
              {{ new Set(books.map((b) => b.author)).size }}
            </strong>
            <small>Authors represented</small>
          </div>
          <div class="stat-card">
            <span>Collection Value</span>
            <strong>
              ${{
                books
                  .reduce((sum, b) => sum + Number(b.price || 0), 0)
                  .toFixed(2)
              }}
            </strong>
            <small>Combined book prices</small>
          </div>
        </section>

        <div class="collection-header">
          <div>
            <h2>Your Collection</h2>
            <p>Browse and manage all your books</p>
          </div>

          <input
            v-model="search"
            class="search-input"
            placeholder="Search books or authors..."
          />
        </div>

        <p v-if="error" class="error-message">
          {{ error }}
        </p>

        <p v-if="loading">Loading books...</p>

        <div v-else-if="filteredBooks.length" class="book-grid">
          <article
            v-for="book in filteredBooks"
            :key="book.id"
            class="book-card"
          >
            <div class="book-cover">
              <img
                v-if="book.imageUrl"
                :src="book.imageUrl"
                :alt="book.title"
                @error="(event) => (event.target.style.display = 'none')"
              />
              <div v-else class="cover-placeholder">B.</div>
            </div>

            <div class="book-details">
              <h3>{{ book.title }}</h3>
              <p>by {{ book.author }}</p>
              <strong>${{ Number(book.price).toFixed(2) }}</strong>

              <div class="book-actions">
                <button @click="editBook(book)">Edit</button>
                <button class="delete-btn" @click="deleteBook(book)">
                  Delete
                </button>
              </div>
            </div>
          </article>
        </div>

        <div v-else class="empty-state">
          <h3>No books found</h3>
          <p v-if="search">Try searching for a different title or author.</p>
          <p v-else>Start your library by adding your first book.</p>
          <button v-if="!search" @click="openAddForm">Add a Book</button>
        </div>
      </section>
    </main>

    <div v-if="showForm" class="modal-overlay" @click.self="showForm = false">
      <div class="modal">
        <div class="modal-header">
          <h2>{{ editingId !== null ? "Edit Book" : "Add New Book" }}</h2>
          <button class="close-btn" @click="showForm = false">×</button>
        </div>

        <form @submit.prevent="saveBook">
          <label>Book Title</label>
          <input
            v-model="form.title"
            placeholder="Enter book title"
            required
            maxlength="200"
          />

          <label>Author</label>
          <input
            v-model="form.author"
            placeholder="Enter author name"
            required
            maxlength="150"
          />

          <label>Price ($)</label>
          <input
            v-model="form.price"
            type="number"
            step="0.01"
            min="0"
            max="99999999.99"
            required
          />

          <label>Cover Image URL (optional)</label>
          <input
            v-model="form.imageUrl"
            placeholder="https://example.com/cover.jpg"
            maxlength="1000"
          />

          <div class="form-actions">
            <button type="button" class="cancel-btn" @click="showForm = false">
              Cancel
            </button>
            <button type="submit" class="primary-btn">
              {{ editingId !== null ? "Save Changes" : "Add Book" }}
            </button>
          </div>
        </form>
      </div>
    </div>
  </div>
</template>
