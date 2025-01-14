<script setup lang="ts">
import { ref, computed, onUnmounted } from 'vue'
import draggable from 'vuedraggable'
import exifr from 'exifr'

interface Photo {
  id: number
  file: File
  url: string
  date: Date
  thumbnail?: string
}

interface Level {
  requiredPhotos: number
  timeLimit: number
  points: number
  name: string
  description: string
}

const levels: Level[] = [
  { 
    requiredPhotos: 3, 
    timeLimit: 60, 
    points: 100,
    name: "Zeitreisender Novize",
    description: "Deine erste Zeitreise beginnt!"
  },
  { 
    requiredPhotos: 4, 
    timeLimit: 50, 
    points: 200,
    name: "Chronologie Entdecker",
    description: "Die Zeit wird knapper!"
  },
  { 
    requiredPhotos: 5, 
    timeLimit: 45, 
    points: 300,
    name: "Zeit Archäologe",
    description: "Grabe tiefer in der Zeitlinie!"
  },
  { 
    requiredPhotos: 6, 
    timeLimit: 40, 
    points: 400,
    name: "Zeitlinien Meister",
    description: "Beweise deine Meisterschaft!"
  },
  { 
    requiredPhotos: 7, 
    timeLimit: 35, 
    points: 500,
    name: "Chronos Champion",
    description: "Die ultimative Herausforderung!"
  }
];

// Jahreszeiten Demo-Bilder
const seasonImages = [
  { 
    url: 'https://images.unsplash.com/photo-1418985991508-e47386d96a71', 
    season: 'Winter',  // Verschneite Winterlandschaft mit Tannen
    date: new Date('2023-12-21')
  },
  { 
    url: 'https://images.unsplash.com/photo-1486944670663-93fc84a4e1fa', 
    season: 'Frühling',  // Kirschblüten und grüne Wiese
    date: new Date('2023-03-21')
  },
  { 
    url: 'https://images.unsplash.com/photo-1473496169904-658ba7c44d8a', 
    season: 'Sommer',  // Sonnenblumenfeld unter blauem Himmel
    date: new Date('2023-06-21')
  },
  { 
    url: 'https://images.unsplash.com/photo-1507371341959-585a53d43f53', 
    season: 'Herbst',  // Bunter Herbstwald mit roten und gelben Blättern
    date: new Date('2023-09-21')
  }
];

const photos = ref<Photo[]>([])
const dragEnabled = ref(true)
const isLoading = ref(false)
const currentLevel = ref(0)
const score = ref(0)
const timeLeft = ref(0)
const gameStarted = ref(false)
const timerInterval = ref<number | null>(null)
const selectedFiles = ref<FileList | null>(null)
const showLevelComplete = ref(false)
const levelScore = ref(0)
const timeBonus = ref(0)
const showTutorial = ref(true)
const showDates = ref(false)
const isMobile = ref(window.innerWidth <= 768)
const gameMode = ref<'custom' | 'seasons'>('custom')

const currentLevelData = computed(() => levels[currentLevel.value])

const progressPercentage = computed(() => {
  return (timeLeft.value / currentLevelData.value.timeLimit) * 100;
})

// Bildoptimierung
async function createThumbnail(file: File): Promise<string> {
  return new Promise((resolve) => {
    const reader = new FileReader();
    reader.onload = (e) => {
      const img = new Image();
      img.onload = () => {
        const canvas = document.createElement('canvas');
        const ctx = canvas.getContext('2d')!;
        
        // Maximale Thumbnail-Größe
        const maxSize = isMobile.value ? 400 : 800;
        let width = img.width;
        let height = img.height;
        
        if (width > height) {
          if (width > maxSize) {
            height = height * (maxSize / width);
            width = maxSize;
          }
        } else {
          if (height > maxSize) {
            width = width * (maxSize / height);
            height = maxSize;
          }
        }
        
        canvas.width = width;
        canvas.height = height;
        
        ctx.drawImage(img, 0, 0, width, height);
        resolve(canvas.toDataURL('image/jpeg', 0.7));
      };
      img.src = e.target?.result as string;
    };
    reader.readAsDataURL(file);
  });
}

const movePhotoLeft = (index: number) => {
  if (index > 0) {
    const temp = photos.value[index];
    photos.value[index] = photos.value[index - 1];
    photos.value[index - 1] = temp;
  }
};

const movePhotoRight = (index: number) => {
  if (index < photos.value.length - 1) {
    const temp = photos.value[index];
    photos.value[index] = photos.value[index + 1];
    photos.value[index + 1] = temp;
  }
};

const formatDateTime = (date: Date) => {
  return date.toLocaleString('de-DE', {
    day: '2-digit',
    month: '2-digit',
    year: 'numeric',
    hour: '2-digit',
    minute: '2-digit'
  });
};

const startTimer = () => {
  if (timerInterval.value) clearInterval(timerInterval.value);
  timeLeft.value = currentLevelData.value.timeLimit;
  timerInterval.value = setInterval(() => {
    if (timeLeft.value > 0) {
      timeLeft.value--;
    } else {
      clearInterval(timerInterval.value!);
      showGameOver();
    }
  }, 1000);
};

const showGameOver = () => {
  const message = `
    🕰️ Zeit abgelaufen!
    Level: ${currentLevelData.value.name}
    Gesamtpunkte: ${score.value}
    
    Versuche es noch einmal!
  `;
  alert(message);
  resetLevel();
};

const resetLevel = () => {
  photos.value.forEach(photo => {
    if (photo.url) URL.revokeObjectURL(photo.url);
  });
  photos.value = [];
  if (timerInterval.value) clearInterval(timerInterval.value);
  timeLeft.value = 0;
  gameStarted.value = false;
  showLevelComplete.value = false;
  selectedFiles.value = null;
};

const handleFileSelect = async (event: Event) => {
  const input = event.target as HTMLInputElement;
  if (!input.files?.length) return;
  
  isLoading.value = true;
  photos.value = [];
  
  try {
    const files = Array.from(input.files);
    const selectedFiles = files.slice(0, currentLevelData.value.requiredPhotos);
    
    for (const file of selectedFiles) {
      try {
        const url = URL.createObjectURL(file);
        const exifData = await exifr.parse(file);
        const date = exifData?.DateTimeOriginal || new Date();
        const thumbnail = await createThumbnail(file);
        
        photos.value.push({
          id: Date.now() + Math.random(),
          file,
          url,
          date,
          thumbnail
        });
      } catch (error) {
        console.error('Error processing file:', error);
      }
    }

    if (photos.value.length === currentLevelData.value.requiredPhotos && !gameStarted.value) {
      gameStarted.value = true;
      startTimer();
    }
  } finally {
    isLoading.value = false;
  }
};

const startSeasonMode = () => {
  gameMode.value = 'seasons';
  photos.value = seasonImages.map((img, index) => ({
    id: index,
    file: new File([], `season-${index}.jpg`),
    url: img.url,
    date: img.date,
    thumbnail: img.url
  }));
  gameStarted.value = true;
  startTimer();
};

const startCustomMode = () => {
  gameMode.value = 'custom';
  resetLevel();
};

const checkOrder = () => {
  const isCorrect = photos.value.every((photo, index) => {
    if (index === 0) return true;
    return photo.date >= photos.value[index - 1].date;
  });
  
  if (isCorrect) {
    timeBonus.value = Math.floor(timeLeft.value * 2);
    levelScore.value = currentLevelData.value.points + timeBonus.value;
    score.value += levelScore.value;
    clearInterval(timerInterval.value!);
    showLevelComplete.value = true;
  } else {
    alert('❌ Falsche Reihenfolge! Versuche es noch einmal.');
    resetLevel();
  }
};

const nextLevel = () => {
  if (currentLevel.value < levels.length - 1) {
    currentLevel.value++;
    resetLevel();
  } else {
    alert(`🏆 Glückwunsch! Du hast alle Level geschafft!\nGesamtpunkte: ${score.value}`);
    currentLevel.value = 0;
    score.value = 0;
    resetLevel();
  }
  showLevelComplete.value = false;
};

const closeTutorial = () => {
  showTutorial.value = false;
};

// Cleanup
onUnmounted(() => {
  if (timerInterval.value) clearInterval(timerInterval.value);
  photos.value.forEach(photo => {
    if (photo.url) URL.revokeObjectURL(photo.url);
  });
});

// Responsive handling
window.addEventListener('resize', () => {
  isMobile.value = window.innerWidth <= 768;
});
</script>

<template>
  <div class="min-h-screen bg-gradient-to-br from-indigo-500 via-purple-500 to-pink-500 p-4">
    <!-- Tutorial Overlay -->
    <div v-if="showTutorial" class="fixed inset-0 bg-black bg-opacity-75 z-50 flex items-center justify-center p-4">
      <div class="bg-white rounded-2xl p-6 max-w-md">
        <h2 class="text-2xl font-bold text-indigo-600 mb-4">🎮 Willkommen zur Foto Timeline Challenge!</h2>
        <ol class="space-y-3 text-gray-700 mb-6">
          <li>1. Wähle zwischen Jahreszeiten-Modus oder eigenen Fotos</li>
          <li>2. Ordne die Bilder nach ihrem Aufnahmedatum</li>
          <li>3. Sei schnell für Extra-Punkte!</li>
          <li>4. Steige Level für Level auf</li>
        </ol>
        <button 
          @click="closeTutorial"
          class="w-full bg-indigo-600 text-white py-3 rounded-xl font-semibold hover:bg-indigo-700 transform hover:scale-105 transition-all duration-200"
        >
          Los geht's! 🚀
        </button>
      </div>
    </div>

    <!-- Level Complete Overlay -->
    <div v-if="showLevelComplete" class="fixed inset-0 bg-black bg-opacity-75 z-50 flex items-center justify-center p-4">
      <div class="bg-white rounded-2xl p-6 max-w-md text-center">
        <h2 class="text-3xl font-bold text-indigo-600 mb-4">🎉 Level Geschafft!</h2>
        <div class="space-y-3 mb-6">
          <p class="text-xl font-semibold text-gray-800">
            Basispunkte: {{ currentLevelData.points }}
          </p>
          <p class="text-lg text-green-600 font-medium">
            Zeitbonus: +{{ timeBonus }}
          </p>
          <p class="text-2xl font-bold text-indigo-600">
            Gesamt: {{ levelScore }}
          </p>
        </div>
        <button 
          @click="nextLevel"
          class="w-full bg-indigo-600 text-white py-3 rounded-xl font-semibold hover:bg-indigo-700 transform hover:scale-105 transition-all duration-200"
        >
          Weiter zum nächsten Level! 🚀
        </button>
      </div>
    </div>

    <!-- Header -->
    <div class="text-center mb-6">
      <h1 class="text-3xl font-bold text-white mb-2">
        📸 Foto Timeline Challenge
      </h1>
      <p class="text-white text-opacity-90">
        {{ currentLevelData.name }}
      </p>
    </div>

    <!-- Game Stats -->
    <div class="bg-white rounded-2xl shadow-xl p-4 mb-6">
      <div class="grid grid-cols-2 gap-4 mb-4">
        <div class="bg-gradient-to-br from-indigo-100 to-purple-100 p-3 rounded-xl text-center">
          <p class="text-sm text-gray-600">Level {{ currentLevel + 1 }}/{{ levels.length }}</p>
          <p class="text-xl font-bold text-indigo-600">{{ currentLevelData.name }}</p>
        </div>
        <div class="bg-gradient-to-br from-purple-100 to-pink-100 p-3 rounded-xl text-center">
          <p class="text-sm text-gray-600">Punkte</p>
          <p class="text-xl font-bold text-purple-600">{{ score }}</p>
        </div>
      </div>

      <!-- Timer Bar -->
      <div class="relative h-4 bg-gray-200 rounded-full overflow-hidden mb-4">
        <div 
          class="absolute top-0 left-0 h-full transition-all duration-1000"
          :class="{
            'bg-green-500': progressPercentage > 66,
            'bg-yellow-500': progressPercentage <= 66 && progressPercentage > 33,
            'bg-red-500': progressPercentage <= 33
          }"
          :style="{ width: `${progressPercentage}%` }"
        ></div>
      </div>
      <p class="text-center font-bold" :class="{
        'text-green-600': progressPercentage > 66,
        'text-yellow-600': progressPercentage <= 66 && progressPercentage > 33,
        'text-red-600': progressPercentage <= 33
      }">
        {{ timeLeft }}s
      </p>

      <!-- Level Info -->
      <div class="text-center mt-4 p-3 bg-gradient-to-r from-indigo-100 to-purple-100 rounded-xl">
        <p class="text-gray-700">
          {{ currentLevelData.description }}
        </p>
        <p class="text-sm text-gray-600 mt-2">
          Ordne {{ currentLevelData.requiredPhotos }} Fotos in {{ currentLevelData.timeLimit }}s
        </p>
      </div>

      <!-- Show Dates Toggle -->
      <div class="mt-4 flex items-center justify-center">
        <label class="flex items-center gap-2 cursor-pointer">
          <input 
            type="checkbox" 
            v-model="showDates" 
            class="w-4 h-4 text-indigo-600 rounded border-gray-300 focus:ring-indigo-500"
          >
          <span class="text-sm text-gray-600">Datum anzeigen</span>
        </label>
      </div>
    </div>

    <!-- Game Mode Selection -->
    <div v-if="!gameStarted" class="mb-6 grid grid-cols-1 md:grid-cols-2 gap-4">
      <button
        @click="startSeasonMode"
        class="p-6 bg-white rounded-2xl shadow-xl hover:bg-indigo-50 transition-colors duration-200 text-center"
      >
        <div class="flex flex-col items-center space-y-4">
          <div class="w-16 h-16 bg-indigo-100 rounded-full flex items-center justify-center">
            <span class="text-3xl">🌞</span>
          </div>
          <div>
            <p class="text-lg font-semibold text-gray-700">Jahreszeiten-Modus</p>
            <p class="text-sm text-gray-500">Ordne die Jahreszeiten richtig zu</p>
          </div>
        </div>
      </button>

      <label
        class="block p-6 bg-white rounded-2xl shadow-xl cursor-pointer hover:bg-indigo-50 transition-colors duration-200 text-center"
      >
        <div class="flex flex-col items-center space-y-4">
          <div class="w-16 h-16 bg-indigo-100 rounded-full flex items-center justify-center">
            <span class="text-3xl">📸</span>
          </div>
          <div>
            <p class="text-lg font-semibold text-gray-700">Eigene Fotos</p>
            <p class="text-sm text-gray-500">Wähle deine eigenen Bilder aus</p>
          </div>
        </div>
        <input 
          type="file" 
          class="hidden" 
          accept="image/*" 
          multiple
          @change="handleFileSelect"
          :disabled="isLoading"
        />
      </label>
    </div>

    <!-- Photos Stack -->
    <div v-if="photos.length > 0" :class="{'flex flex-col gap-4': isMobile, 'mt-4 mb-28': !isMobile}">
      <draggable
        v-model="photos"
        :disabled="!dragEnabled"
        item-key="id"
        :class="{
          'flex flex-col gap-4': isMobile,
          'overflow-x-auto snap-x snap-mandatory flex gap-4 pb-6': !isMobile
        }"
        ghost-class="opacity-50"
      >
        <template #item="{ element, index }">
          <div :class="{
            'w-full': isMobile,
            'snap-center shrink-0 first:ml-[calc(50%-160px)] last:mr-[calc(50%-160px)]': !isMobile
          }">
            <div class="bg-white rounded-xl shadow-xl overflow-hidden transform transition-all duration-300 relative group"
                 :class="{ 'w-80': !isMobile }">
              <!-- Position Badge -->
              <div class="absolute top-2 left-2 bg-black/70 text-white px-3 py-1 rounded-full text-sm z-10">
                Position: {{ index + 1 }}/{{ photos.length }}
              </div>

              <!-- Control Buttons -->
              <div class="absolute top-2 right-2 flex gap-2 z-10">
                <button
                  @click="movePhotoLeft(index)"
                  :disabled="index === 0"
                  class="bg-black/70 text-white w-8 h-8 rounded-full flex items-center justify-center hover:bg-black/90 disabled:opacity-50 disabled:cursor-not-allowed"
                >
                  <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 19l-7-7 7-7" />
                  </svg>
                </button>
                <button
                  @click="movePhotoRight(index)"
                  :disabled="index === photos.length - 1"
                  class="bg-black/70 text-white w-8 h-8 rounded-full flex items-center justify-center hover:bg-black/90 disabled:opacity-50 disabled:cursor-not-allowed"
                >
                  <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7" />
                  </svg>
                </button>
              </div>

              <div class="relative">
                <img 
                  :src="element.thumbnail || element.url" 
                  :alt="gameMode === 'seasons' ? `${element.season} Bild` : `Photo ${element.id}`"
                  class="w-full object-cover"
                  :class="{ 'h-56': !isMobile, 'h-48': isMobile }"
                  draggable="false"
                >
                <div class="absolute inset-0 bg-gradient-to-t from-black/60 to-transparent"></div>
                <div class="absolute bottom-0 left-0 right-0 p-3 text-white">
                  <p v-if="showDates" class="font-medium mb-1 flex items-center gap-2">
                    <span class="bg-black/50 px-2 py-1 rounded-lg">
                      📅 {{ formatDateTime(element.date) }}
                    </span>
                  </p>
                  <p class="text-sm opacity-75" v-if="index === 0">
                    {{ isMobile ? 'Ziehe oder nutze die Pfeile zum Sortieren' : 'Ziehe oder nutze die Pfeile zum Sortieren' }}
                  </p>
                </div>
              </div>
            </div>
          </div>
        </template>
      </draggable>

      <!-- Mobile Instructions -->
      <div class="text-white text-center opacity-75 mt-4" v-if="!isMobile">
        <div class="flex flex-col items-center justify-center gap-2">
          <div class="flex items-center gap-2">
            <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 7h12m0 0l-4-4m4 4l-4 4m0 6H4m0 0l4 4m-4-4l4-4" />
            </svg>
            <span class="text-sm">Wische zum Scrollen</span>
          </div>
          <div class="flex items-center gap-2">
            <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M7 16V4m0 0L3 8m4-4l4 4m6 0v12m0 0l4-4m-4 4l-4-4" />
            </svg>
            <span class="text-sm">Ziehe die Bilder oder nutze die Pfeile zum Sortieren</span>
          </div>
        </div>
      </div>
    </div>

    <!-- Check Button -->
    <button
      v-if="photos.length > 0"
      @click="checkOrder"
      class="fixed bottom-4 left-4 right-4 max-w-lg mx-auto py-4 bg-gradient-to-r from-indigo-600 to-purple-600 text-white text-lg font-semibold rounded-xl hover:from-indigo-700 hover:to-purple-700 transform hover:scale-[1.02] transition-all duration-200 shadow-lg"
    >
      Reihenfolge prüfen 🎯
    </button>
  </div>
</template>

<style scoped>
.active-card {
  cursor: grab;
  touch-action: none;
  will-change: transform;
}
.active-card:active {
  cursor: grabbing;
}

/* Smooth Scrolling für die Foto-Liste */
.snap-x {
  scroll-behavior: smooth;
  -webkit-overflow-scrolling: touch;
  scrollbar-width: none; /* Firefox */
}

.snap-x::-webkit-scrollbar {
  display: none; /* Chrome, Safari, Edge */
}

/* Verbesserte Touch-Bedienung */
@media (hover: none) {
  .snap-x {
    touch-action: pan-x pinch-zoom;
  }
}
</style>