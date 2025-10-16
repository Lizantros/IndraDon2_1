<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'

// --- NO IMPORTS OF OTHER COMPONENTS HERE ---

declare interface Post {
  _id?: string
  post_name: string
  post_content: string
  attributes: {
    creation_date: any
  }
}

const storage = ref<any>()
const postsData = ref<Post[]>([])

const initDatabase = () => {
  console.log('=> Connexion à la base de données')
  const db = new PouchDB('http://admin:de4Dke032e!@127.0.0.1:5984/database')
  if (db) {
    console.log('Connecté à la collection : ' + db?.name)
    storage.value = db
  } else {
    console.warn('Echec lors de la connexion à la base de données')
  }
}

const fetchData = (): any => {
  storage.value
    .allDocs({ include_docs: true })
    .then((result: any) => {
      console.log('=> Données récupérées :', result.rows)
      postsData.value = result.rows.map((row: any) => row.doc)
    })
    .catch((error: any) => {
      console.error('=> Erreur lors de la récupération des données :', error)
    })
}

const addDocument = () => {
  if (!storage.value) return
  const newDoc = {
    post_name: 'Mon Nouveau Post',
    post_content: `Créé à ${new Date().toLocaleTimeString()}`,
    attributes: {
      creation_date: new Date().toISOString(),
    },
  }
  storage.value
    .post(newDoc)
    .then((response: any) => {
      console.log('✅ Document ajouté avec succès!', response.id)
      fetchData() // Refresh the list
    })
    .catch((err: any) => {
      console.error("❌ Erreur lors de l'ajout", err)
    })
}

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
  fetchData()
})
</script>

<template>
  <h1>Fetch Data</h1>

  <button @click="addDocument">Ajouter un Document</button>

  <hr />

  <article v-for="post in postsData" :key="post._id">
    <h2>{{ post.post_name }}</h2>
    <p>{{ post.post_content }}</p>
  </article>
</template>

<style scoped>
button {
  margin-bottom: 1rem;
}
</style>
