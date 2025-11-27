<script lang="ts">
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'

PouchDB.plugin(PouchDBFind)

export default {
  data() {
    return {
      db: null as any,
      remoteDB: null as any,
      documents: [] as any[],
      syncHandler: null as any,
      isOffline: true,
      searchQuery: '',
    }
  },

  mounted() {
    this.initDB()
  },

  methods: {
    initDB() {
      this.db = new PouchDB('ma_db_locale')
      this.remoteDB = new PouchDB('http://admin:de4Dke032e!@localhost:5984/ma_db')

      this.db
        .createIndex({
          index: { fields: ['title'] },
        })
        .then(() => {})
        .catch((err: any) => {
          console.log('Erreur créa index', err)
        })

      this.recupererTousLesDocs()
    },

    toggleMode() {
      if (this.isOffline) {
        this.isOffline = false
        this.syncHandler = this.db
          .sync(this.remoteDB, {
            live: true,
            retry: true,
          })
          .on('change', (info: any) => {
            console.log(info)
            if (this.searchQuery === '') this.recupererTousLesDocs()
          })
          .on('error', (err: any) => {
            console.error(err)
          })
      } else {
        this.isOffline = true
        if (this.syncHandler) {
          this.syncHandler.cancel()
          this.syncHandler = null
        }
      }
    },

    factory() {
      const docs = []
      for (let i = 1; i <= 10; i++) {
        docs.push({
          _id: new Date().toISOString() + '_' + i,
          title: `document Factory ${i}`,
          content: `contenu gen auto ${i}`,
        })
      }
      this.db
        .bulkDocs(docs)
        .then(() => {
          this.recupererTousLesDocs()
        })
        .catch((err: any) => console.log(err))
    },

    rechercher() {
      if (!this.searchQuery) {
        this.recupererTousLesDocs()
        return
      }

      this.db
        .find({
          selector: {
            title: { $regex: RegExp(this.searchQuery, 'i') },
          },
        })
        .then((result: any) => {
          this.documents = result.docs
        })
        .catch((err: any) => {
          console.log(err)
        })
    },

    recupererTousLesDocs() {
      this.db
        .allDocs({ include_docs: true })
        .then((result: any) => {
          this.documents = result.rows
            .map((row: any) => row.doc)
            .filter((doc: any) => !doc._id.startsWith('_design')) // c'est pour les modifs pour les dcs qui commencent par _
        })
        .catch((err: any) => {
          console.log(err)
        })
    },

    genererObjetDemo() {
      const timestamp = Date.now()
      return {
        _id: 'local_doc_' + timestamp,
        title: 'doc local ' + timestamp,
        content: 'contenu créer en local ' + timestamp,
      }
    },

    ajouterDoc() {
      const nouveauDoc = this.genererObjetDemo()
      this.db
        .put(nouveauDoc)
        .then(() => {
          this.recupererTousLesDocs()
        })
        .catch((err: any) => console.log(err))
    },

    modifDoc(doc: any) {
      this.db
        .get(doc._id)
        .then((docActuel: any) => {
          docActuel.content = docActuel.content + ' (Modifié)'
          return this.db.put(docActuel)
        })
        .then(() => {
          this.recupererTousLesDocs()
        })
        .catch((err: any) => console.log(err))
    },

    supprimerDoc(doc: any) {
      this.db
        .remove(doc._id, doc._rev)
        .then(() => {
          this.recupererTousLesDocs()
        })
        .catch((error: any) => console.log(error))
    },
  },
}
</script>

<template>
  <div style="padding: 20px">
    <div style="margin-bottom: 20px; border-bottom: 2px solid #eee; padding-bottom: 20px">
      <button
        @click="toggleMode"
        :style="{ backgroundColor: isOffline ? 'salmon' : 'lightgreen' }"
        style="padding: 10px; font-weight: bold"
      >
        {{ isOffline ? 'Mode HL(se connecter)' : 'Mode L (se déco)' }}
      </button>

      <div style="margin-top: 10px">
        <button @click="ajouterDoc">Ajouter un doc</button>
        <button @click="factory" style="margin-left: 10px">Facto</button>
      </div>
    </div>

    <div style="margin-bottom: 20px; background: #f9f9f9; padding: 10px">
      <h4>Recherche</h4>
      <input
        type="text"
        v-model="searchQuery"
        placeholder="Rechercher nom"
        style="padding: 5px; width: 200px"
      />
      <button @click="rechercher">Rechercher</button>
      <button
        @click="
          () => {
            searchQuery = ''
            recupererTousLesDocs()
          }
        "
      >
        Reset
      </button>
    </div>

    <h3>Documents ({{ documents.length }})</h3>
    <div v-if="documents.length === 0">rien trouvé</div>

    <div
      v-for="doc in documents"
      :key="doc._id"
      style="border: 1px solid #ccc; margin: 10px 0; padding: 10px"
    >
      <p><strong>ID:</strong> {{ doc._id }}</p>
      <p><strong>Titre:</strong> {{ doc.title }}</p>
      <p><strong>Contenu:</strong> {{ doc.content }}</p>
      <button @click="modifDoc(doc)">Modif</button>
      <button @click="supprimerDoc(doc)">Suppr</button>
    </div>
  </div>
</template>
