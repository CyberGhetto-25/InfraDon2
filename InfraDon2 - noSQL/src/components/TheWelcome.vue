<script setup lang="ts">
import { ref, onMounted } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'

// plugin pour les requetes "find"
PouchDB.plugin(PouchDBFind)

// type de doc dans la bd (avec ajout des comm et likes)
interface CommentDoc {
  _id?: string
  _rev?: string
  type: 'comment'
  messageId: string
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
}

// refs pour la bd + ui
const messagesDB = ref<any>(null)
const commentsDB = ref<any>(null)
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
  console.log('init base infradon2 (messages + comments)')

  // --- MESSAGES
  const localMessages = new PouchDB('test_infradon2_messages_local')
  const remoteMessages = new PouchDB('http://admin:admin@127.0.0.1:5984/test_infradon2_messages')

  localMessages.replicate
    .from(remoteMessages)
    .on('complete', () => console.log('replicate ok messages (remote -> local)'))
    .on('error', (err: any) => console.error('err replicate messages', err))

  const syncMessages = localMessages.sync(remoteMessages, { live: true, retry: true })
  syncMessages
    .on('change', () => fetchData())
    .on('error', (err: any) => console.error('sync messages err', err))

  // --- COMMENTS
  const localComments = new PouchDB('test_infradon2_comments_local')
  const remoteComments = new PouchDB('http://admin:admin@127.0.0.1:5984/test_infradon2_comments')

  localComments.replicate
    .from(remoteComments)
    .on('complete', () => console.log('replicate ok comments (remote -> local)'))
    .on('error', (err: any) => console.error('err replicate comments', err))

  const syncComments = localComments.sync(remoteComments, { live: true, retry: true })
  syncComments
    .on('change', () => fetchData())
    .on('error', (err: any) => console.error('sync comments err', err))

  // handlers (pour offline toggle)
  syncHandler.value = { syncMessages, syncComments }

  // refs accessibles partout
  messagesDB.value = localMessages
  commentsDB.value = localComments

  createIndexes()
  fetchData()
}

// recup de tous les docs
const fetchData = async () => {
  if (!messagesDB.value || !commentsDB.value) return

  try {
    // messages
    const res = await messagesDB.value.allDocs({ include_docs: true })
    const all = res.rows
      .filter((r: any) => r.doc && !String(r.id).startsWith('_design/'))
      .map((r: any) => r.doc)

    docs.value = all.filter((d: any) => d.type === 'document') as InfradonDoc[]

    // pour chaque message -> charger ses commentaires
    for (const d of docs.value) {
      if (!d._id) continue
      await loadCommentsForMessage(d._id)
    }

    console.log('nb docs:', docs.value.length)
  } catch (err) {
    console.error('err fetchData', err)
  }
}

const commentsByMessage = ref<Record<string, CommentDoc[]>>({})

const getComments = (messageId?: string) => {
  if (!messageId) return []
  return commentsByMessage.value[messageId] ?? []
}

const loadCommentsForMessage = async (messageId: string) => {
  if (!commentsDB.value) return

  try {
    const res = await commentsDB.value.find({
      selector: { type: 'comment', messageId },
      sort: ['created_at']
    })
    commentsByMessage.value[messageId] = res.docs as CommentDoc[]
  } catch (err) {
    try {
      const res2 = await commentsDB.value.find({
        selector: { type: 'comment', messageId }
      })
      commentsByMessage.value[messageId] = res2.docs as CommentDoc[]
    } catch (e2) {
      console.error('err loadCommentsForMessage', err, e2)
    }
  }
}

// creation d un doc
const createDoc = () => {
  if (!messagesDB.value) return

  const newDoc: InfradonDoc = {
    type: 'document',
    title: formTitle.value || ('doc ' + Date.now()),
    description: formDescription.value || 'test auto infradon2',
    created_at: new Date().toISOString(),
    status: 'draft',
    author: 'Marc Bridy',
    likes: 0
  }

  messagesDB.value
    .post(newDoc)
    .then(() => {
      formTitle.value = ''
      formDescription.value = ''
      fetchData()
    })
    .catch((err: any) => console.error('err createDoc', err))
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
  if (!messagesDB.value || !selectedDoc.value) {
    console.warn('pas de doc sélectionné')
    return
  }

  const updatedDoc: InfradonDoc = {
    ...selectedDoc.value,
    title: formTitle.value || selectedDoc.value.title,
    description: formDescription.value || selectedDoc.value.description,
    status: 'updated'
  }

  messagesDB.value
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
  if (!messagesDB.value) return
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

    await messagesDB.value.remove(doc._id, doc._rev)
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
  try {
    // --- messages
    if (messagesDB.value) {
      await messagesDB.value.createIndex({ index: { fields: ['title'] } })
      await messagesDB.value.createIndex({ index: { fields: ['likes'] } })
      await messagesDB.value.createIndex({ index: { fields: ['type'] } })
    }

    // --- comments
    if (commentsDB.value) {
      await commentsDB.value.createIndex({ index: { fields: ['type'] } })
      await commentsDB.value.createIndex({ index: { fields: ['messageId'] } })
      await commentsDB.value.createIndex({ index: { fields: ['messageId', 'created_at'] } })
      await commentsDB.value.createIndex({ index: { fields: ['created_at'] } })
    }

    console.log('index ok (messages + comments)')
  } catch (err) {
    console.error('err createIndexes', err)
  }
}


// recherche par titre (pouchdb-find)
const searchDocs = async () => {
  if (!messagesDB.value) return

  const term = searchTerm.value.trim()

  // si input vide -> on recharge tout
  if (!term) {
    fetchData()
    return
  }

  try {
    const res = await messagesDB.value.find({
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
  if (!messagesDB.value || !doc._id || !doc._rev) return

  const updated: InfradonDoc = {
    ...doc,
    likes: (doc.likes || 0) + 1
  }

  try {
    await messagesDB.value.put(updated)
    fetchData()
  } catch (err) {
    console.error('err likeDoc', err)
  }
}

const addComment = async (doc: InfradonDoc) => {
  if (!commentsDB.value || !doc._id) return

  const key = doc._id
  const txt = commentDrafts.value[key] || ''
  if (!txt.trim()) return

  const newComment: CommentDoc = {
    type: 'comment',
    messageId: doc._id,
    text: txt.trim(),
    author: 'anonyme',
    created_at: new Date().toISOString()
  }

  try {
    await commentsDB.value.post(newComment)
    commentDrafts.value[key] = ''
    await loadCommentsForMessage(doc._id)
  } catch (err) {
    console.error('err addComment', err)
  }
}


// suppr un commmentaire
const deleteComment = async (comment: CommentDoc) => {
  if (!commentsDB.value || !comment._id || !comment._rev) return

  try {
    await commentsDB.value.remove(comment._id, comment._rev)
    await loadCommentsForMessage(comment.messageId)
  } catch (err) {
    console.error('err deleteComment', err)
  }
}

// edit commentaire
const editComment = async (comment: CommentDoc) => {
  if (!commentsDB.value || !comment._id || !comment._rev) return

  const nv = window.prompt('nouveau texte du commentaire', comment.text)
  if (nv === null) return

  const updated: CommentDoc = {
    ...comment,
    text: nv
  }

  try {
    await commentsDB.value.put(updated)
    await loadCommentsForMessage(comment.messageId)
  } catch (err) {
    console.error('err editComment', err)
  }
}


// tri par nb de likes
const sortByLikesDesc = async () => {
  if (!messagesDB.value) return

  try {
    const res = await messagesDB.value.find({
      selector: { likes: { $gte: 0 }, type: 'document' }
      // pas de sort ici si ça casse, sinon on trie en JS
    })

    const arr = res.docs as InfradonDoc[]
    arr.sort((a, b) => (b.likes || 0) - (a.likes || 0))
    docs.value = arr
  } catch (err) {
    console.error('err sortByLikesDesc', err)
  }
}


// factory pour générer plein de doc random
const generateFakeDocs = async (nb: number) => {
  if (!messagesDB.value) return

  for (let i = 0; i < nb; i++) {
    const rand = Math.random().toString(36).substring(2, 7)

    await messagesDB.value.post({
      type: 'document',
      title: 'doc ' + rand,
      description: 'description auto ' + rand,
      created_at: new Date().toISOString(),
      status: 'auto',
      author: 'N/A',
      likes: 0,
    })
  }
  fetchData()
}


// focntion toggle offline/online
const toggleOffline = async () => {
  // si on est ONLINE -> passer OFFLINE
  if (!isOffline.value) {
    console.log('passage en offline')

    if (syncHandler.value?.syncMessages) syncHandler.value.syncMessages.cancel()
    if (syncHandler.value?.syncComments) syncHandler.value.syncComments.cancel()
    syncHandler.value = null

    isOffline.value = true
    return
  }

  // si on est OFFLINE -> retour ONLINE
  console.log('retour en online')
  initDatabase()
  isOffline.value = false
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

          <div v-if="getComments(doc._id).length">
            <p><strong>Commentaires:</strong></p>

            <section v-for="first in getComments(doc._id).slice(0, 1)" :key="first._id">
              <h4>Premier commentaire</h4>
              <blockquote>
                <p>{{ first.text }}</p>
                <footer>
                  {{ first.author }}
                  ·
                  {{ new Date(first.created_at).toLocaleString() }}
                </footer>
              </blockquote>
            </section>

            <details>
              <summary>Tous les commentaires ({{ getComments(doc._id).length }})</summary>
              <ul>
                <li v-for="c in getComments(doc._id)" :key="c._id">
                  <span>{{ c.text }}</span>
                  <small> ({{ c.author }})</small>
                  <button type="button" @click="editComment(c)">Modifier</button>
                  <button type="button" @click="deleteComment(c)">Supprimer</button>
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
