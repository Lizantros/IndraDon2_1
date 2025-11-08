<script lang="ts">
import PouchDB from 'pouchdb'

export default {
  data() {
    return {
      db: null as any,
      remoteDB: null as any,
      documents: [] as any[],
    }
  },

  mounted() {
    this.initDB()
  },

  methods: {
    initDB() {
      this.db = new PouchDB('ma_db_locale')
      this.remoteDB = new PouchDB('http://admin:de4Dke032e!@localhost:5984/ma_db')

      this.recupererTousLesDocs()
    },

    synchroniser() {
      this.db
        .sync(this.remoteDB)
        .on('change', (info: any) => {
          console.log('Changement détecté during sync', info)
          this.recupererTousLesDocs()
        })
        .on('complete', (info: any) => {
          console.log('Synchro terminée !', info)
          this.recupererTousLesDocs()
          alert('Synchronisation terminée !')
        })
        .on('error', (err: any) => {
          console.error('Erreur de synchro', err)
        })
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
        _id: 'local_doc_' + timestamp,
        title: 'Doc Local ' + timestamp,
        content: 'Contenu créé localement ' + timestamp,
      }
    },

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

    modifDoc(doc: any) {
      this.db
        .get(doc._id)
        .then((docActuel: any) => {
          docActuel.content = docActuel.content + ' (Modifié)'
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
  <div style="padding: 20px">
    <div style="margin-bottom: 20px">
      <button @click="ajouterDoc">1. Ajouter un doc (Local)</button>
      <button @click="synchroniser" style="margin-left: 10px; background-color: lightblue">
        2. 🔄 Synchroniser avec le serveur
      </button>
    </div>

    <h3>Documents (Base Locale)</h3>
    <div v-if="documents.length === 0">Aucun document local.</div>

    <div
      v-for="doc in documents"
      :key="doc._id"
      style="border: 1px solid #ccc; margin: 10px 0; padding: 10px"
    >
      <p><strong>ID:</strong> {{ doc._id }}</p>
      <p><strong>Titre:</strong> {{ doc.title }}</p>
      <p><strong>Contenu:</strong> {{ doc.content }}</p>
      <button @click="modifDoc(doc)">Modif Local</button>
      <button @click="supprimerDoc(doc)">Suppr Local</button>
    </div>
  </div>
</template>
