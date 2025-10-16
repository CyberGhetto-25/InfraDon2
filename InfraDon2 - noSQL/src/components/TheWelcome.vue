<script setup lang="ts">
import { onMounted, ref } from 'vue';
import PouchDB from 'pouchdb'

// === Types adaptés à ton JSON ===
interface Character {
  name: string;
  age: number;
  affiliation: string;
  lightsaber: boolean;
}

interface Planet {
  name: string;
  population: string;
  affiliation: string;
}

interface SWDoc {
  _id: string;
  _rev: string;
  characters: Character[];
  planets: Planet[];
}

// Référence à la base de données
const storage = ref<any>(null)
// Données stockées (docs complets)
const postsData = ref<SWDoc[]>([])

// Initialisation de la base de données
const initDatabase = () => {
  console.log('=> Connexion à la base de données');
  const db = new PouchDB('http://admin:admin@localhost:5984/test_infradon2')
  if (db) {
    console.log('Connecté à la collection : ' + db.name)
    storage.value = db
  } else {
    console.warn('Échec lors de la connexion à la base de données')
  }
}

// Récupération des données
const fetchData = () => {
  if (!storage.value) {
    console.warn('⚠️ Base non initialisée')
    return
  }

  storage.value
    .allDocs({ include_docs: true })
    .then((res: any) => {
      const docs: SWDoc[] = res.rows
        .filter((row: any) => row.doc && !String(row.id).startsWith('_design/'))
        .map((row: any) => row.doc as SWDoc)

      postsData.value = docs
      console.log('✅ Données récupérées :', postsData.value)
    })
    .catch((err: any) => {
      console.error('❌ Erreur lors du fetch des données :', err)
    })
}

onMounted(() => {
  console.log('=> Composant initialisé');
  initDatabase()
  fetchData()
});
</script>

<template>
  <h1>Fetch Data</h1>

  <section v-for="doc in postsData" :key="doc._id" style="margin-bottom: 1.5rem;">
    <h2>Document: {{ doc._id }}</h2>

    <h3>Characters</h3>
    <ul>
      <li v-for="c in doc.characters" :key="c.name">
        <strong>{{ c.name }}</strong> — age: {{ c.age }},
        affiliation: {{ c.affiliation }},
        lightsaber: {{ c.lightsaber ? 'yes' : 'no' }}
      </li>
    </ul>

    <h3>Planets</h3>
    <ul>
      <li v-for="p in doc.planets" :key="p.name">
        <strong>{{ p.name }}</strong> — population: {{ p.population }},
        affiliation: {{ p.affiliation }}
      </li>
    </ul>
  </section>
</template>
