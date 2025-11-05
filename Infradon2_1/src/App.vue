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

    genererObjetDemo() {
      const timestamp = Date.now()
      return {
        _id: 'le meilleur nom de l univers... oui j en ai marre' + timestamp,
        title: 'Doc ' + timestamp,
        content: 'cest super fake ' + timestamp,
      }
    },

    //https://pouchdb.com/api.html#create_document
    ajouterDoc() {
      const nouveauDoc = this.genererObjetDemo()

      this.db
        .put(nouveauDoc)
        .then((reponse: any) => {
          console.log(reponse)
          this.recupererTousLesDocs()
        })
        .catch((erreur: any) => {
          console.log(erreur)
        })
    },

    // https://pouchdb.com/api.html#fetch_document
    modifDoc(doc: any) {
      this.db
        .get(doc._id)
        .then((docActuel: any) => {
          docActuel.content = docActuel.content + ' , modif le : ' + new Date().toLocaleString() // Rajouter la date de la modif pour une dose légendaire de fun YAY

          return this.db.put(docActuel)
        })
        .then((resultat: any) => {
          console.log(resultat)
          this.recupererTousLesDocs()
        })
        .catch((err: any) => {
          console.log(err)
        })
    },

    // https://pouchdb.com/api.html#delete_document
    supprimerDoc(doc: any) {
      this.db
        .remove(doc._id, doc._rev)
        .then((res: any) => {
          console.log(res)
          this.recupererTousLesDocs()
        })
        .catch((error: any) => {
          console.log(error)
        })
    },
  },
}
</script>

<template>
  <div>
    <button @click="ajouterDoc">Ajouter un doc</button>

    <div v-for="doc in documents" :key="doc._id">
      <p><strong>ID:</strong> {{ doc._id }}</p>
      <p><strong>Titre:</strong> {{ doc.title }}</p>
      <p><strong>Contenu:</strong> {{ doc.content }}</p>
      <button @click="modifDoc(doc)">Modif</button>
      <button @click="supprimerDoc(doc)">Suppr</button>
      <hr />
    </div>
  </div>
</template>
