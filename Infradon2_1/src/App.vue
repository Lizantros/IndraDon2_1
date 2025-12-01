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
      nouveauTitre: '',
      nouveauContenu: '',
      selectedCategory: '',
      categories: [] as any[],
      nouvelleCategorie: '',
      nouveauCommentaireInput: {} as Record<string, string>
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
          index: { fields: ['title', 'type'] },
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
      docs.push({ _id: new Date().toISOString() + '_cat1', type: 'category', name: 'cat1' })
      docs.push({ _id: new Date().toISOString() + '_cat2', type: 'category', name: 'cat2' })

      for (let i = 1; i <= 10; i++) {
        docs.push({
          _id: new Date().toISOString() + '_' + i,
          type: 'post',
          title: `Post Factory ${i}`,
          content: `contenu gen auto ${i}`,
          category: 'cat1',
          likes: 0,
          comments: []
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
            type: 'post',
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
          const tous = result.rows
            .map((row: any) => row.doc)
            .filter((doc: any) => !doc._id.startsWith('_design'))
          this.documents = tous.filter((doc: any) => doc.type === 'post' || !doc.type)
          this.categories = tous.filter((doc: any) => doc.type === 'category')
        })
        .catch((err: any) => {
          console.log(err)
        })
    },

    ajouterCategorie() {
      if (!this.nouvelleCategorie) return
      const cat = {
        _id: new Date().toISOString() + '_cat',
        type: 'category',
        name: this.nouvelleCategorie
      }
      this.db.put(cat).then(() => {
        this.nouvelleCategorie = ''
        this.recupererTousLesDocs()
      }).catch((err: any) => console.log(err))
    },

    ajouterDoc() {
      const nouveauDoc = {
        _id: new Date().toISOString(),
        type: 'post',
        category: this.selectedCategory,
        title: this.nouveauTitre,
        content: this.nouveauContenu,
        likes: 0,
        comments: []
      }
      this.db
        .put(nouveauDoc)
        .then(() => {
          this.nouveauTitre = ''
          this.nouveauContenu = ''
          this.selectedCategory = ''
          this.recupererTousLesDocs()
        })
        .catch((err: any) => console.log(err))
    },

    likerPost(doc: any) {
      this.db
        .get(doc._id)
        .then((docActuel: any) => {
          docActuel.likes = (docActuel.likes || 0) + 1
          return this.db.put(docActuel)
        })
        .then(() => {
          this.recupererTousLesDocs()
        })
        .catch((err: any) => console.log(err))
    },

    ajouterCommentaire(doc: any) {
      const texte = this.nouveauCommentaireInput[doc._id]
      if (!texte) return

      this.db
        .get(doc._id)
        .then((docActuel: any) => {
          if (!docActuel.comments) docActuel.comments = []
          docActuel.comments.push({
            text: texte,
            likes: 0
          })
          return this.db.put(docActuel)
        })
        .then(() => {
          this.nouveauCommentaireInput[doc._id] = ''
          this.recupererTousLesDocs()
        })
        .catch((err: any) => console.log(err))
    },

    likerCommentaire(doc: any, index: number) {
      this.db
        .get(doc._id)
        .then((docActuel: any) => {
          if (docActuel.comments && docActuel.comments[index]) {
             docActuel.comments[index].likes = (docActuel.comments[index].likes || 0) + 1
          }
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

      <div style="margin-top: 10px; border: 1px solid #ccc; padding: 10px;">
        <div style="margin-bottom: 10px; padding-bottom: 10px; border-bottom: 1px dashed #ccc">
          <input v-model="nouvelleCategorie" placeholder="Nouvelle Catégorie">
          <button @click="ajouterCategorie">Ajouter cat</button>
        </div>

        <input v-model="nouveauTitre" placeholder="Titre" style="margin-right: 5px;">
        <input v-model="nouveauContenu" placeholder="Contenu" style="margin-right: 5px;">

        <select v-model="selectedCategory">
            <option value="">Caté</option>
            <option v-for="c in categories" :key="c._id" :value="c.name">{{ c.name }}</option>
        </select>

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
      <p><strong>Titre:</strong> {{ doc.title }} <span v-if="doc.category">({{ doc.category }})</span></p>
      <p><strong>Contenu:</strong> {{ doc.content }}</p>

      <div style="margin-bottom: 10px;">
        <button @click="likerPost(doc)">Like Doc ({{ doc.likes || 0 }})</button>
        <button @click="supprimerDoc(doc)" style="margin-left: 5px;">Suppr</button>
      </div>

      <div style="background: #eee; padding: 10px; margin-top: 10px;">
        <strong>Commentaires:</strong>
        <div v-for="(com, idx) in doc.comments" :key="idx" style="border-bottom: 1px solid #ccc; padding: 5px;">
            {{ com.text }}
            <button @click="likerCommentaire(doc, idx)" style="font-size: 0.8em; margin-left: 5px;">Like Com ({{ com.likes || 0 }})</button>
        </div>

        <div style="margin-top: 5px;">
            <input v-model="nouveauCommentaireInput[doc._id]" placeholder="Nouveau commentaire">
            <button @click="ajouterCommentaire(doc)">Ajouter Com</button>
        </div>
      </div>
    </div>
  </div>
</template>
