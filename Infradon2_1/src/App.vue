<script lang="ts">
import PouchDB from 'pouchdb'

export default {
  data() {
    return {
      db: null as any,
      documents: [] as any[],
    }
  },

  mounted() {
    this.initDB()
  },

  methods: {
    //https://pouchdb.com/api.html#create_database
    initDB() {
      this.db = new PouchDB('http://admin:de4Dke032e!@localhost:5984/ma_db')
      this.recupererTousLesDocs()
    },

    recupererTousLesDocs() {
      this.db
        .allDocs({ include_docs: true })
        .then((result: any) => {
          this.documents = result.rows.map((row: any) => row.doc)
        })
        .catch((err: any) => {
          console.log(err)
        })
    },
  },
}
</script>

<template>
  <div>
    <div v-for="doc in documents" :key="doc._id">
      <p><strong>ID:</strong> {{ doc._id }}</p>
      <p><strong>Titre:</strong> {{ doc.title }}</p>
      <p><strong>Contenu:</strong> {{ doc.content }}</p>
      <hr />
    </div>
  </div>
</template>
