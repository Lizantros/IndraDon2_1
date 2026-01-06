<script lang="ts">
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'

PouchDB.plugin(PouchDBFind)

declare interface Comment {
  _id: string
  _rev?: string
  type: string
  post_id: string
  text: string
  likes: number
}

declare interface Post {
  _id: string
  _rev?: string
  type: string
  post_name: string     
  post_content: string  
  category?: string
  attributes?: {
    creation_date: any
  }
  likes?: number
  virtual_comments?: Comment[]
}

export default {
  data() {
    return {
      db: null as any,
      remoteDB: null as any,
      dbComments: null as any,
      remoteDBComments: null as any,

      posts: [] as Post[],
      categories: [] as any[],

      syncHandler: null as any,
      syncCommentsHandler: null as any,
      isOffline: true,

      query: '',
      inputTitle: '',
      inputBody: '',
      inputCatName: '',
      selectedCat: '',
      draftComments: {} as Record<string, string>,
    }
  },

  mounted() {
    this.initDB()
  },

  methods: {
    initDB() {
      this.db = new PouchDB('ma_db_posts_loic_peyramaure')
      this.dbComments = new PouchDB('ma_db_comments_loic_peyramaure')

      this.remoteDB = new PouchDB('http://admin:de4Dke032e!@localhost:5984/ma_db_posts_loic_peyramaure')
      this.remoteDBComments = new PouchDB('http://admin:de4Dke032e!@localhost:5984/ma_db_comments_loic_peyramaure')

      // Création des index (requis pour find et sort)
      this.db.createIndex({ index: { fields: ['type'] } })
        .then(() => {
          return this.db.createIndex({ index: { fields: ['post_name'] } })
        })
        .then(() => {
          // Index requis pour le tri par likes
          return this.db.createIndex({ index: { fields: ['likes'] } })
        })
        .then(() => this.fetchData())
        .catch((err: any) => console.log(err))
        
      this.dbComments.createIndex({ index: { fields: ['post_id'] } })
        .catch((err: any) => console.log(err))
    },

    toggleSync() {
      if (this.isOffline) {
        this.isOffline = false
        const opts = { live: true, retry: true }
        
        this.syncHandler = this.db.sync(this.remoteDB, opts)
          .on('change', () => this.fetchData())
        
        this.syncCommentsHandler = this.dbComments.sync(this.remoteDBComments, opts)
          .on('change', () => this.fetchData())
      } else {
        this.isOffline = true
        if (this.syncHandler) this.syncHandler.cancel()
        if (this.syncCommentsHandler) this.syncCommentsHandler.cancel()
        this.syncHandler = null
        this.syncCommentsHandler = null
      }
    },

    fetchData() {
      const selectorPosts = { type: 'post' }
      const selectorCats = { type: 'category' }

      this.db.find({ selector: selectorCats })
        .then((resCats: any) => {
          this.categories = resCats.docs
          return this.db.find({ selector: selectorPosts })
        })
        .then((resPosts: any) => {
          const loadedPosts = resPosts.docs as Post[]
          
          return this.dbComments.allDocs({ include_docs: true })
            .then((resComments: any) => {
               const loadedComments = resComments.rows
                 .map((row: any) => row.doc)
                 .filter((d: any) => !d._id.startsWith('_design'))

               loadedPosts.forEach((p: Post) => {
                 p.virtual_comments = loadedComments.filter((c: Comment) => c.post_id === p._id)
               })

               this.posts = loadedPosts
            })
        })
        .catch((err: any) => console.log(err))
    },

    generateFakeData() {
      const timestamp = new Date().toISOString()
      const batchPosts: any[] = []
      const batchComments: any[] = []

      batchPosts.push({ _id: timestamp + '_cat1', type: 'category', name: 'Général' })
      batchPosts.push({ _id: timestamp + '_cat2', type: 'category', name: 'Technique' })

      for (let i = 1; i <= 5; i++) {
        const pid = timestamp + '_post_' + i
        batchPosts.push({
          _id: pid,
          type: 'post',
          post_name: 'Post généré ' + i,
          post_content: 'Ceci est le contenu du post numéro ' + i,
          category: 'Général',
          likes: 0,
          attributes: { creation_date: new Date().toISOString() }
        })
        
        batchComments.push({
          _id: pid + '_c1',
          type: 'comment',
          post_id: pid,
          text: 'Commentaire test ' + i,
          likes: 0,
        })
      }

      this.db.bulkDocs(batchPosts)
        .then(() => this.dbComments.bulkDocs(batchComments))
        .then(() => this.fetchData())
        .catch((err: any) => console.log(err))
    },

    search() {
      if (!this.query) {
        this.fetchData()
        return
      }
      this.db.find({
        selector: {
          type: 'post',
          post_name: { $regex: RegExp(this.query, 'i') },
        },
      })
      .then((result: any) => {
        const foundPosts = result.docs as Post[]
        this.dbComments.allDocs({ include_docs: true }).then((resCom: any) => {
            const allComs = resCom.rows.map((r: any) => r.doc)
            foundPosts.forEach(p => {
               p.virtual_comments = allComs.filter((c: any) => c.post_id === p._id)
            })
            this.posts = foundPosts
          })
      })
      .catch((err: any) => console.log(err))
    },

    topLikes() {
      this.db.find({
        selector: {
          type: 'post',
          likes: { $gte: 0 } 
        },
        sort: [{ 'likes': 'desc' }], 
        limit: 10
      })
      .then((result: any) => {
        const foundPosts = result.docs as Post[]
        this.dbComments.allDocs({ include_docs: true }).then((resCom: any) => {
            const allComs = resCom.rows.map((r: any) => r.doc)
            foundPosts.forEach(p => {
               p.virtual_comments = allComs.filter((c: any) => c.post_id === p._id)
            })
            this.posts = foundPosts
          })
      })
      .catch((err: any) => console.log(err))
    },

    addCategory() {
      if (!this.inputCatName) return
      const cat = { 
        _id: new Date().toISOString() + '_cat', 
        type: 'category', 
        name: this.inputCatName 
      }
      this.db.put(cat)
        .then(() => { 
          this.inputCatName = ''
          this.fetchData() 
        })
        .catch((err: any) => console.log(err))
    },

    addPost() {
      const doc: Post = {
        _id: new Date().toISOString(),
        type: 'post',
        category: this.selectedCat,
        post_name: this.inputTitle,
        post_content: this.inputBody,
        likes: 0,
        attributes: { 
          creation_date: new Date().toISOString() 
        }
      }
      this.db.put(doc)
        .then(() => {
          this.inputTitle = ''
          this.inputBody = ''
          this.selectedCat = ''
          this.fetchData()
        })
        .catch((err: any) => console.log(err))
    },

    likePost(doc: Post) {
      this.db.get(doc._id)
        .then((curr: any) => {
          curr.likes = (curr.likes || 0) + 1
          return this.db.put(curr)
        })
        .then(() => this.fetchData())
        .catch((err: any) => console.log(err))
    },

    deletePost(doc: Post) {
      if (!doc._rev) return
      this.db.remove(doc._id, doc._rev)
        .then(() => this.fetchData())
        .catch((err: any) => console.log(err))
    },

    addComment(post: Post) {
      const txt = this.draftComments[post._id]
      if (!txt) return
      const com: Comment = {
        _id: new Date().toISOString() + '_com',
        type: 'comment',
        post_id: post._id,
        text: txt,
        likes: 0,
      }
      this.dbComments.put(com)
        .then(() => {
          this.draftComments[post._id] = ''
          this.fetchData()
        })
        .catch((err: any) => console.log(err))
    },

    likeComment(post: Post, index: number) {
      if (!post.virtual_comments?.[index]) return
      const comId = post.virtual_comments[index]._id
      this.dbComments.get(comId)
        .then((c: any) => {
          c.likes = (c.likes || 0) + 1
          return this.dbComments.put(c)
        })
        .then(() => this.fetchData())
        .catch((err: any) => console.log(err))
    },
  },
}
</script>

<template>
  <div class="container">
    
    <div class="section header">
      <h2>TP Infra Données 2 - Loïc Peyramaure</h2>
      
      <button @click="toggleSync" :class="isOffline ? 'btn-red' : 'btn-green'">
        Statut: {{ isOffline ? 'OFFLINE (Local)' : 'ONLINE (Sync)' }}
      </button>
      
      <div style="margin-top: 10px;">
        <button @click="generateFakeData">Générer Données de test</button>
      </div>
    </div>

    <div class="section forms">
      <div class="form-group">
        <label>Nouvelle Catégorie :</label>
        <input v-model="inputCatName" placeholder="Nom..." />
        <button @click="addCategory">Ajouter Catégorie</button>
      </div>

      <div class="form-group" style="border-top: 1px dashed #ccc; padding-top: 10px;">
        <label>Nouveau Post :</label>
        <div style="display: flex; gap: 5px; flex-wrap: wrap;">
            <input v-model="inputTitle" placeholder="Titre (post_name)" />
            <input v-model="inputBody" placeholder="Contenu (post_content)" style="flex-grow: 1;" />
            
            <select v-model="selectedCat">
            <option value="">-- Catégorie --</option>
            <option v-for="c in categories" :key="c._id" :value="c.name">{{ c.name }}</option>
            </select>
            
            <button @click="addPost">Publier</button>
        </div>
      </div>
    </div>

    <div class="section search">
      <input type="text" v-model="query" placeholder="Recherche par nom..." />
      <button @click="search">Chercher</button>
      <button @click="topLikes">Top 10 Likes</button>
      <button @click="() => { query = ''; fetchData() }">Reset</button>
    </div>

    <div class="post-list">
      <h3>Liste des Posts ({{ posts.length }})</h3>

      <div v-for="p in posts" :key="p._id" class="post-item">
        <div class="post-header">
          <strong>{{ p.post_name }}</strong> 
          <span v-if="p.category" style="font-size: 0.8em; color: #666;">[{{ p.category }}]</span>
          
          <button @click="deletePost(p)" style="float: right;">Supprimer</button>
        </div>

        <p class="post-content">{{ p.post_content }}</p>
        
        <div class="post-actions">
           Likes: {{ p.likes || 0 }} 
           <button @click="likePost(p)">Like</button>
        </div>

        <div class="comments-box">
          <div style="font-weight: bold; margin-bottom: 5px;">Commentaires :</div>
          
          <div v-for="(com, idx) in p.virtual_comments" :key="idx" class="comment-item">
            - {{ com.text }} (Likes: {{ com.likes || 0 }})
            <button @click="likeComment(p, idx)" style="font-size: 0.8em;">Like</button>
          </div>

          <div style="margin-top: 5px;">
            <input v-model="draftComments[p._id]" placeholder="Votre commentaire" />
            <button @click="addComment(p)">Ajouter</button>
          </div>
        </div>

      </div>
    </div>

  </div>
</template>

<style>
.container {
  font-family: sans-serif;
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  background-color: #f4f4f4; 
  color: #333333; 
  min-height: 100vh;
}

input, select, textarea {
  background-color: #ffffff;
  color: #000000;
  border: 1px solid #999;
  padding: 5px;
}

.section {
  border: 1px solid #ccc;
  padding: 15px;
  margin-bottom: 20px;
  background-color: #ffffff; 
  color: #000000; 
}

h2, h3 {
  margin-top: 0;
  color: #000000;
}

button {
  cursor: pointer;
  margin-right: 5px;
  padding: 5px 10px;
  background-color: #e0e0e0;
  color: #000;
  border: 1px solid #999;
}

.btn-red {
  background-color: #ffcccc;
  border: 1px solid red;
  color: #cc0000;
}

.btn-green {
  background-color: #ccffcc;
  border: 1px solid green;
  color: #006600;
}

.form-group {
  margin-bottom: 10px;
}

.post-item {
  border: 1px solid #999;
  padding: 15px;
  margin-bottom: 15px;
  background-color: #ffffff; 
  color: #000000; 
}

.post-header {
  border-bottom: 1px solid #eee;
  padding-bottom: 5px;
  margin-bottom: 10px;
  display: flex;
  justify-content: space-between;
}

.post-content {
  margin: 10px 0;
  white-space: pre-wrap; 
}

.comments-box {
  background-color: #f0f0f0; 
  border: 1px solid #ddd;
  padding: 10px;
  margin-top: 15px;
  color: #333;
}

.comment-item {
  border-bottom: 1px dashed #ccc;
  padding: 5px 0;
  display: flex;
  justify-content: space-between;
  align-items: center;
}
</style>