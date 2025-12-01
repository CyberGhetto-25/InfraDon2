<script setup lang="ts">
import { ref, onMounted } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'

// plugin pour les requetes "find"
PouchDB.plugin(PouchDBFind)

// type de doc dans la bd (avec ajout des comm et likes)
interface CommentDoc {
  id: string
  text: string
  author: string
  created_at: string
}

interface InfradonDoc {
  _id?: string
  _rev?: string
  type?: 'document'
  title: string
  description: string
  created_at: string
  status: string
  author: string
  likes?: number
  comments?: CommentDoc[]
}

// category
interface CategoryDoc {
  _id?: string
  _rev?: string
  type: 'category'
  name: string
  created_at: string
}

// refs pour la bd + ui
const storage = ref<any>(null) // osef des types generiques ici
const docs = ref<InfradonDoc[]>([])
const formTitle = ref('')
const formDescription = ref('')
const categories = ref<CategoryDoc[]>([])
const newCategoryName = ref('')
const selectedDoc = ref<InfradonDoc | null>(null)
const deletingIds = ref<Set<string>>(new Set())
const searchTerm = ref('')
const isOffline = ref(false)
const syncHandler = ref<any>(null) // pour garder le sync et pouvoir le cancel
const commentDrafts = ref<{ [key: string]: string }>({})

// init de la base (local + remote + sync)
const initDatabase = () => {
  console.log('init base infradon2')

  // base locale dans le navigateur
  const localDB = new PouchDB('test_infradon2_local')

  // base distante couchdb
  const remoteDB = new PouchDB('http://admin:admin@127.0.0.1:5984/test_infradon2')

  // premier "pull" remote -> local
  localDB
    .replicate
    .from(remoteDB)
    .on('complete', () => {
      console.log('replicate ok (remote -> local)')
    })
    .on('error', (err: any) => {
      console.error('err replicate', err)
    })

  // sync en continu local <-> remote
  const sync = localDB.sync(remoteDB, { live: true, retry: true })

  sync
    .on('change', (info: any) => {
      console.log('sync change', info)
      fetchData()
    })
    .on('paused', (err: any) => {
      if (err) console.warn('sync paused', err)
    })
    .on('active', () => {
      console.log('sync active')
    })
    .on('error', (err: any) => {
      console.error('sync err', err)
    })

  syncHandler.value = sync

  // on garde la base locale pour la suite
  storage.value = localDB
  createIndexes()
}

// recup de tous les docs
const fetchData = () => {
  if (!storage.value) return

  storage.value
    .allDocs({ include_docs: true })
    .then((res: any) => {
      const allDocs = res.rows
        .filter((r: any) => r.doc && !String(r.id).startsWith('_design/'))
        .map((r: any) => r.doc as any)

      docs.value = allDocs.filter(
        (d: any) => !d.type || d.type === 'document'
      ) as InfradonDoc[]

      categories.value = allDocs.filter(
        (d: any) => d.type === 'category'
      ) as CategoryDoc[]

      console.log('nb docs:', docs.value.length, ' | nb categories:', categories.value.length)
    })
    .catch((err: any) => {
      console.error('err fetchData', err)
    })
}

// creation d un doc
const createDoc = () => {
  if (!storage.value) return

  const newDoc: InfradonDoc = {
    type: 'document',
    title: formTitle.value || ('doc ' + Date.now()),
    description: formDescription.value || 'test auto infradon2',
    created_at: new Date().toISOString(),
    status: 'draft',
    author: 'Marc Bridy',
    likes: 0,
    comments: []
  }

  storage.value
    .post(newDoc)
    .then(() => {
      formTitle.value = ''
      formDescription.value = ''
      fetchData()
    })
    .catch((err: any) => {
      console.error('err createDoc', err)
    })
}

// selection d un doc pour edition
const selectDoc = (doc: InfradonDoc) => {
  // on copie pour pas modifier direct
  selectedDoc.value = { ...doc }
  formTitle.value = doc.title
  formDescription.value = doc.description
}

// maj d un doc
const updateDoc = () => {
  if (!storage.value || !selectedDoc.value) {
    console.warn('pas de doc sélectionné')
    return
  }

  const updatedDoc: InfradonDoc = {
    ...selectedDoc.value,
    title: formTitle.value || selectedDoc.value.title,
    description: formDescription.value || selectedDoc.value.description,
    status: 'updated'
  }

  storage.value
    .put(updatedDoc)
    .then(() => {
      selectedDoc.value = null
      formTitle.value = ''
      formDescription.value = ''
      fetchData()
    })
    .catch((err: any) => {
      console.error('err updateDoc', err)
    })
}

// suppression d un doc
const deleteDoc = async (doc: InfradonDoc) => {
  if (!storage.value) return
  if (!doc._id || !doc._rev) {
    console.warn('doc sans id/rev => pas delete')
    return
  }

  const ok = window.confirm(`supprimer « ${doc.title} » ?`)
  if (!ok) return

  try {
    deletingIds.value.add(doc._id)

    // si on etait en train d editer ce doc la
    if (selectedDoc.value && selectedDoc.value._id === doc._id) {
      selectedDoc.value = null
      formTitle.value = ''
      formDescription.value = ''
    }

    await storage.value.remove(doc._id, doc._rev)
    console.log('doc supprimé:', doc._id)
    await fetchData()
  } catch (err) {
    console.error('err deleteDoc', err)
    alert('erreur suppression, voir console')
  } finally {
    deletingIds.value.delete(doc._id)
  }
}

// creation index pour la recherche
const createIndexes = async () => {
  if (!storage.value) return

  try {
    await storage.value.createIndex({
      index: {
        fields: ['title']
      }
    })
    await storage.value.createIndex({
      index: {
        fields: ['likes']
      }
    })
    await storage.value.createIndex({
      index: {
        fields: ['type']
      }
    })
    console.log('index title/likes/type ok')
  } catch (err) {
    console.error('err createIndexes', err)
  }
}

// recherche par titre (pouchdb-find)
const searchDocs = async () => {
  if (!storage.value) return

  const term = searchTerm.value.trim()

  // si input vide -> on recharge tout
  if (!term) {
    fetchData()
    return
  }

  try {
    const res = await storage.value.find({
      selector: {
        title: { $regex: new RegExp(term, 'i') }
      },
      sort: ['title']
    })

    docs.value = res.docs as InfradonDoc[]
    console.log('res recherche:', docs.value.length)
  } catch (err) {
    console.error('err searchDocs', err)
  }
}
// incérmenter le compteur de likes
const likeDoc = async (doc: InfradonDoc) => {
  if (!storage.value || !doc._id || !doc._rev) return

  const updated: InfradonDoc = {
    ...doc,
    likes: (doc.likes || 0) + 1
  }

  try {
    await storage.value.put(updated)
    fetchData()
  } catch (err) {
    console.error('err likeDoc', err)
  }
}

const addComment = async (doc: InfradonDoc) => {
  if (!storage.value || !doc._id || !doc._rev) return

  const key = doc._id
  const txt = commentDrafts.value[key] || ''

  if (!txt.trim()) return

  const newComment: CommentDoc = {
    id: Math.random().toString(36).substring(2, 9),
    text: txt,
    author: 'anonyme', // on a pas de gestion des users
    created_at: new Date().toISOString()
  }

  const oldComments = doc.comments || []

  const updated: InfradonDoc = {
    ...doc,
    comments: [...oldComments, newComment]
  }

  try {
    await storage.value.put(updated)
    commentDrafts.value[key] = ''
    fetchData()
  } catch (err) {
    console.error('err addComment', err)
  }
}

// suppr un commmentaire
const deleteComment = async (doc: InfradonDoc, commentId: string) => {
  if (!storage.value || !doc._id || !doc._rev) return

  const oldComments = doc.comments || []
  const newComments = oldComments.filter(c => c.id !== commentId)

  const updated: InfradonDoc = {
    ...doc,
    comments: newComments
  }

  try {
    await storage.value.put(updated)
    fetchData()
  } catch (err) {
    console.error('err deleteComment', err)
  }
}

// edit commentaire
const editComment = async (doc: InfradonDoc, comment: CommentDoc) => {
  if (!storage.value || !doc._id || !doc._rev) return

  const nv = window.prompt('nouveau texte du commentaire', comment.text)
  if (nv === null) return

  const oldComments = doc.comments || []
  const newComments = oldComments.map(c =>
    c.id === comment.id ? { ...c, text: nv } : c
  )

  const updated: InfradonDoc = {
    ...doc,
    comments: newComments
  }

  try {
    await storage.value.put(updated)
    fetchData()
  } catch (err) {
    console.error('err editComment', err)
  }
}

// tri par nb de likes
const sortByLikesDesc = async () => {
  if (!storage.value) return

  try {
    const res = await (storage.value as any).find({
      selector: {
        likes: { $gte: 0 }
      },
      sort: [{ likes: 'desc' }]
    })

    docs.value = res.docs as InfradonDoc[]
  } catch (err) {
    console.error('err sortByLikesDesc', err)
  }
}

// factory pour générer plein de doc random
const generateFakeDocs = async (nb: number) => {
  if (!storage.value) return

  for (let i = 0; i < nb; i++) {
    const rand = Math.random().toString(36).substring(2, 7)

    await storage.value.post({
      type: 'document',
      title: 'doc ' + rand,
      description: 'description auto ' + rand,
      created_at: new Date().toISOString(),
      status: 'auto',
      author: 'N/A',
      likes: 0,
      comments: []
    })
  }
  fetchData()
}
// ajoute une catégory
const createCategory = async () => {
  if (!storage.value) return

  const name = newCategoryName.value.trim()
  if (!name) return

  const cat: CategoryDoc = {
    type: 'category',
    name,
    created_at: new Date().toISOString()
  }

  try {
    await storage.value.post(cat)
    newCategoryName.value = ''
    fetchData()
  } catch (err) {
    console.error('err createCategory', err)
  }
}

// suppression d une catégorie
const deleteCategory = async (cat: CategoryDoc) => {
  if (!storage.value || !cat._id || !cat._rev) return

  const ok = window.confirm(`supprimer la catégorie « ${cat.name} » ?`)
  if (!ok) return

  try {
    await storage.value.remove(cat._id, cat._rev)
    fetchData()
  } catch (err) {
    console.error('err deleteCategory', err)
  }
}

// focntion toggle offline/online
const toggleOffline = async () => {
  if (!storage.value) return

  if (!isOffline.value) {
    console.log('passage en offline')

    if (syncHandler.value) {
      syncHandler.value.cancel()
      syncHandler.value = null
    }

    isOffline.value = true
  } else {
    console.log('retour en online')

    const remoteDB = new PouchDB('http://admin:admin@127.0.0.1:5984/test_infradon2')

    await storage.value.replicate.from(remoteDB)

    const sync = storage.value.sync(remoteDB, { live: true, retry: true })

    sync
      .on('change', (info: any) => {
        console.log('sync change', info)
        fetchData()
      })
      .on('paused', (err: any) => {
        if (err) console.warn('sync paused', err)
      })
      .on('active', () => console.log('sync active'))
      .on('error', (err: any) => console.error('sync err', err))

    syncHandler.value = sync
    isOffline.value = false
  }
}

// reset champ de recherche + liste
const resetSearch = () => {
  searchTerm.value = ''
  fetchData()
}

// init au chargement du composant
onMounted(() => {
  console.log('infradon2 mounted')
  initDatabase()
  fetchData()
})
</script>

<template>
  <main class="container">
    <h1>Gestion des documents Infradon2</h1>

    <button type="button" @click="toggleOffline">
      {{ isOffline ? 'offline' : 'online' }}
    </button>

    <!-- recherche -->
    <section>
      <h2>Recherche par titre</h2>
      <form @submit.prevent>
        <div class="grid">
          <label>
            Titre
            <input type="search" v-model="searchTerm" @input="searchDocs" placeholder="tape un titre ou une partie" />
          </label>

          <div>
            <label>&nbsp;</label>
            <div class="grid">
              <button type="button" @click="resetSearch">
                Réinitialiser
              </button>
              <button type="button" @click="sortByLikesDesc">
                Trier par like
              </button>
            </div>
          </div>
        </div>
      </form>
    </section>

    <!-- Button Factory -->
    <section>
      <button type="button" @click="generateFakeDocs(20)">
        Générer 20 documents
      </button>
    </section>

    <!-- formulaire -->
    <section>
      <h2>{{ selectedDoc ? 'Modifier un document' : 'Créer un document' }}</h2>

      <form @submit.prevent="selectedDoc ? updateDoc() : createDoc()">
        <label>
          Titre
          <input type="text" v-model="formTitle" placeholder="titre" />
        </label>

        <label>
          Description
          <textarea v-model="formDescription" placeholder="description" rows="3"></textarea>
        </label>

        <div class="grid">
          <button type="submit">
            {{ selectedDoc ? 'Mettre à jour' : 'Créer' }}
          </button>

          <button v-if="selectedDoc" type="button" @click="selectedDoc = null; formTitle = ''; formDescription = ''">
            Annuler
          </button>
        </div>
      </form>
    </section>

    <section>
      <h2>Catégories</h2>

      <form @submit.prevent="createCategory">
        <label>
          Nom de la catégorie
          <input type="text" v-model="newCategoryName" placeholder="ex: urgent, marketing, etc." />
        </label>
        <button type="submit">
          Ajouter
        </button>
      </form>

      <ul v-if="categories.length">
        <li v-for="cat in categories" :key="cat._id">
          {{ cat.name }}
          <small>
            · créé le {{ new Date(cat.created_at).toLocaleString() }}
          </small>
          <button type="button" @click="deleteCategory(cat)">
            Supprimer
          </button>
        </li>
      </ul>

      <p v-else>Aucune catégorie pour l’instant.</p>
    </section>

    <!-- liste des docs -->
    <section>
      <h2>Liste des documents</h2>

      <div class="docs-grid" v-if="docs.length">
        <article v-for="doc in docs" :key="doc._id">
          <header>
            <h3>{{ doc.title }}</h3>
            <p>{{ doc.description }}</p>
            <small>
              auteur: {{ doc.author }} |
              status: {{ doc.status }} |
              créé le: {{ new Date(doc.created_at).toLocaleString() }}
            </small>
          </header>

          <p>
            <button type="button" @click="likeDoc(doc)">
              Like ({{ doc.likes || 0 }})
            </button>
          </p>

          <div v-if="doc.comments && doc.comments.length">
            <p><strong>Commentaires:</strong></p>

            <!-- premier commentaire -->
            <section>
              <h4>Premier commentaire</h4>
              <blockquote>
                <p>{{ doc.comments![0].text }}</p>
                <footer>
                  {{ doc.comments![0].author }}
                  ·
                  {{ new Date(doc.comments![0].created_at).toLocaleString() }}
                </footer>
              </blockquote>
            </section>

            <details>
              <summary>Tous les commentaires ({{ doc.comments.length }})</summary>
              <ul>
                <li v-for="c in doc.comments" :key="c.id">
                  <span>{{ c.text }}</span>
                  <small> ({{ c.author }})</small>
                  <button type="button" @click="editComment(doc, c)">
                    Modifier
                  </button>
                  <button type="button" @click="deleteComment(doc, c.id)">
                    Supprimer
                  </button>
                </li>
              </ul>
            </details>
          </div>

          <div>
            <form @submit.prevent="addComment(doc)">
              <label>
                Commentaire
                <input type="text" :placeholder="'Ajouter un commentaire sur ' + doc.title"
                  v-model="commentDrafts[doc._id!]" />
              </label>
              <button type="submit">
                Envoyer
              </button>
            </form>
          </div>

          <footer class="grid">
            <button type="button" @click="selectDoc(doc)">
              Modifier
            </button>

            <button type="button" @click="deleteDoc(doc)" :disabled="deletingIds.has(doc._id!)">
              {{ deletingIds.has(doc._id!) ? 'Suppression...' : 'Supprimer' }}
            </button>
          </footer>
        </article>
      </div>

      <p v-else>Aucun document pour l’instant.</p>
    </section>
  </main>
</template>
