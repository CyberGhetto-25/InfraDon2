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
  title: string
  description: string
  created_at: string
  status: string
  author: string
  likes?: number
  comments?: CommentDoc[]
}

// refs pour la bd + ui
const storage = ref<any>(null) // osef des types generiques ici
const docs = ref<InfradonDoc[]>([])
const formTitle = ref('')
const formDescription = ref('')
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
      docs.value = res.rows
        .filter((r: any) => r.doc && !String(r.id).startsWith('_design/'))
        .map((r: any) => r.doc as InfradonDoc)
      console.log('nb docs:', docs.value.length)
    })
    .catch((err: any) => {
      console.error('err fetchData', err)
    })
}

// creation d un doc
const createDoc = () => {
  if (!storage.value) return

  const newDoc: InfradonDoc = {
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
    console.log('index title ok')
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
  <div style="padding: 2rem;">
    <h1>Gestion des documents Infradon2</h1>

    <button @click="toggleOffline" style="padding:0.4rem 0.8rem;margin-bottom:1rem;">
      {{ isOffline ? 'offline' : 'online' }}
    </button>
    <!-- recherche -->
    <section style="margin-bottom: 1.5rem;">
      <h2>Recherche par titre</h2>
      <input type="text" v-model="searchTerm" @input="searchDocs" placeholder="tape un titre ou une partie"
        style="display:block;margin:0.5rem 0;padding:0.5rem;width:300px;" />
      <button @click="resetSearch" style="padding:0.3rem 0.8rem;">
        Réinitialiser
      </button>
      <button @click="sortByLikesDesc" style="padding:0.3rem 0.8rem;margin-left:0.5rem;">
        Trier par like
      </button>
    </section>

    <!-- Button Factory -->
    <button @click="generateFakeDocs(20)" style="padding:0.4rem 0.8rem;margin-top:1rem;background:#444;color:white;">
      Générer 20 documents
    </button>

    <!-- formulaire -->
    <section style="margin-top: 1.5rem; margin-bottom: 2rem;">
      <h2>{{ selectedDoc ? 'Modifier un document' : 'Créer un document' }}</h2>

      <input type="text" v-model="formTitle" placeholder="titre"
        style="display:block;margin:0.5rem 0;padding:0.5rem;width:300px;" />

      <textarea v-model="formDescription" placeholder="description"
        style="display:block;margin:0.5rem 0;padding:0.5rem;width:300px;height:80px;"></textarea>

      <button @click="selectedDoc ? updateDoc() : createDoc()" style="padding:0.5rem 1rem;margin-right:0.5rem;">
        {{ selectedDoc ? 'Mettre à jour' : 'Créer' }}
      </button>

      <button v-if="selectedDoc" @click="
        selectedDoc = null;
      formTitle = '';
      formDescription = '';
      " style="padding:0.5rem 1rem;">
        Annuler
      </button>
    </section>

    <!-- liste des docs -->
    <section>
      <h2>Liste des documents</h2>
      <div v-for="doc in docs" :key="doc._id"
        style="border:1px solid #555;border-radius:8px;padding:1rem;margin-bottom:1rem;">
        <h3>{{ doc.title }}</h3>
        <p>{{ doc.description }}</p>
        <p>
          <small>
            auteur: {{ doc.author }} |
            status: {{ doc.status }} |
            créé le: {{ new Date(doc.created_at).toLocaleString() }}
          </small>
        </p>

        <!-- like -->
        <p style="margin:0.5rem 0;">
          <button @click="likeDoc(doc)" style="padding:0.2rem 0.6rem;margin-right:0.5rem;">
            Like ({{ doc.likes || 0 }})
          </button>
        </p>

        <!-- commentaires existants -->
        <div v-if="doc.comments && doc.comments.length" style="margin-top:0.5rem;">
          <p><strong>Commentaires:</strong></p>
          <ul>
            <li v-for="c in doc.comments" :key="c.id" style="margin-bottom:0.3rem;">
              <span>{{ c.text }}</span>
              <small> ({{ c.author }})</small>
              <button @click="editComment(doc, c)" style="margin-left:0.5rem;padding:0.1rem 0.4rem;">
                Modifier
              </button>
              <button @click="deleteComment(doc, c.id)" style="margin-left:0.3rem;padding:0.1rem 0.4rem;">
                Supprimer
              </button>
            </li>
          </ul>
        </div>

        <!-- ajout d un com -->
        <div style="margin-top:0.5rem;">
          <input type="text" :placeholder="'Ajouter un commentaire sur ' + doc.title" v-model="commentDrafts[doc._id!]"
            style="width:100%;padding:0.3rem;margin-bottom:0.3rem;" />
          <button @click="addComment(doc)" style="padding:0.3rem 0.8rem;">
            Envoyer
          </button>
        </div>

        <!-- boutons doc -->
        <div style="margin-top:0.8rem;">
          <button @click="selectDoc(doc)" style="padding:0.3rem 0.8rem;margin-right:0.4rem;">
            Modifier
          </button>

          <button @click="deleteDoc(doc)" :disabled="deletingIds.has(doc._id!)" style="padding:0.3rem 0.8rem;">
            {{ deletingIds.has(doc._id!) ? 'Suppression...' : 'Supprimer' }}
          </button>
        </div>
      </div>
    </section>
  </div>
</template>
