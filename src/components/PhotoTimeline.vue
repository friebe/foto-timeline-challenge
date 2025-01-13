<script setup lang="ts">
import { ref, computed, onUnmounted } from 'vue'
import draggable from 'vuedraggable'
import exifr from 'exifr'

interface Photo {
  id: number
  file: File
  url: string
  date: Date
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

const currentLevelData = computed(() => levels[currentLevel.value])

const progressPercentage = computed(() => {
  return (timeLeft.value / currentLevelData.value.timeLimit) * 100;
})

// Neue Funktion zur Bildoptimierung
const optimizeImage = async (file: File): Promise<{ url: string, date: Date }> => {
  // EXIF-Daten zuerst extrahieren, da wir das Original-Bild dafür brauchen
  const exifData = await exifr.parse(file);
  const date = exifData?.DateTimeOriginal || new Date();

  // Bild in Canvas laden und optimieren
  const img = new Image();
  const canvas = document.createElement('canvas');
  const ctx = canvas.getContext('2d');

  return new Promise((resolve) => {
    img.onload = () => {
      // Maximale Dimensionen festlegen
      const MAX_WIDTH = 800;
      const MAX_HEIGHT = 600;
      
      let width = img.width;
      let height = img.height;

      // Proportionales Verkleinern
      if (width > height) {
        if (width > MAX_WIDTH) {
          height *= MAX_WIDTH / width;
          width = MAX_WIDTH;
        }
      } else {
        if (height > MAX_HEIGHT) {
          width *= MAX_HEIGHT / height;
          height = MAX_HEIGHT;
        }
      }

      // Canvas-Größe setzen
      canvas.width = width;
      canvas.height = height;

      // Bild zeichnen
      ctx?.drawImage(img, 0, 0, width, height);

      // Als JPEG mit reduzierter Qualität exportieren
      const optimizedUrl = canvas.toDataURL('image/jpeg', 0.7);

      // Speicher freigeben
      URL.revokeObjectURL(img.src);
      resolve({ url: optimizedUrl, date });
    };

    // Temporäre URL für das Originalbild
    img.src = URL.createObjectURL(file);
  });
};

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
  photos.value = [];
  if (timerInterval.value) clearInterval(timerInterval.value);
  timeLeft.value = 0;
  gameStarted.value = false;
  showLevelComplete.value = false;
  selectedFiles.value = null;
};

const handleFolderSelect = async (event: Event) => {
  const input = event.target as HTMLInputElement;
  if (!input.files?.length) return;
  
  selectedFiles.value = input.files;
  await loadPhotosForCurrentLevel();
};

// Optimierte Version der loadPhotosForCurrentLevel Funktion
const loadPhotosForCurrentLevel = async () => {
  if (!selectedFiles.value) return;
  
  isLoading.value = true;
  photos.value = [];
  
  try {
    const files = Array.from(selectedFiles.value);
    const shuffledFiles = files
      .filter(file => file.type.startsWith('image/')) // Nur Bilder akzeptieren
      .sort(() => Math.random() - 0.5);
    
    const levelFiles = shuffledFiles.slice(0, currentLevelData.value.requiredPhotos);
    
    // Bilder in Batches verarbeiten
    const BATCH_SIZE = 2;
    for (let i = 0; i < levelFiles.length; i += BATCH_SIZE) {
      const batch = levelFiles.slice(i, i + BATCH_SIZE);
      const optimizedBatch = await Promise.all(
        batch.map(async (file) => {
          try {
            const { url, date } = await optimizeImage(file);
            return {
              id: Date.now() + Math.random(),
              file,
              url,
              date
            };
          } catch (error) {
            console.error('Error processing file:', error);
            return null;
          }
        })
      );
      
      // Erfolgreiche Ergebnisse zum Array hinzufügen
      photos.value.push(...optimizedBatch.filter(Boolean) as Photo[]);
    }

    if (photos.value.length === currentLevelData.value.requiredPhotos && !gameStarted.value) {
      gameStarted.value = true;
      startTimer();
    }
  } finally {
    isLoading.value = false;
  }
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
    loadPhotosForCurrentLevel();
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

// Aufräumen beim Komponentenabbau
onUnmounted(() => {
  if (timerInterval.value) clearInterval(timerInterval.value);
  // Alle URLs freigeben
  photos.value.forEach(photo => {
    if (photo.url.startsWith('blob:')) {
      URL.revokeObjectURL(photo.url);
    }
  });
});
</script>

<template>
  <div class="min-h-screen bg-gradient-to-br from-indigo-500 via-purple-500 to-pink-500 p-4">
    <!-- Tutorial Overlay -->
    <div v-if="showTutorial" class="fixed inset-0 bg-black bg-opacity-75 z-50 flex items-center justify-center p-4">
      <div class="bg-white rounded-2xl p-6 max-w-md">
        <h2 class="text-2xl font-bold text-indigo-600 mb-4">🎮 Willkommen zur Foto Timeline Challenge!</h2>
        <ol class="space-y-3 text-gray-700 mb-6">
          <li>1. Wähle einen Ordner mit deinen Fotos</li>
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

    <!-- Folder Selection -->
    <div v-if="!gameStarted" class="mb-6">
      <label
        class="block w-full p-6 bg-white rounded-2xl shadow-xl cursor-pointer hover:bg-indigo-50 transition-colors duration-200 text-center"
      >
        <div class="flex flex-col items-center space-y-4">
          <div class="w-16 h-16 bg-indigo-100 rounded-full flex items-center justify-center">
            <svg class="w-8 h-8 text-indigo-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z" />
            </svg>
          </div>
          <div>
            <p class="text-lg font-semibold text-gray-700">Wähle deinen Foto-Ordner</p>
            <p class="text-sm text-gray-500">Klicke hier um zu starten</p>
          </div>
        </div>
        <input 
          type="file" 
          class="hidden" 
          accept="image/*" 
          multiple
          webkitdirectory
          @change="handleFolderSelect"
          :disabled="isLoading"
        />
      </label>
    </div>

    <!-- Photos Stack -->
    <div v-if="photos.length > 0" class="mt-4 mb-28">
      <draggable
        v-model="photos"
        :disabled="!dragEnabled"
        item-key="id"
        class="overflow-x-auto snap-x snap-mandatory flex gap-4 pb-6"
        ghost-class="opacity-50"
      >
        <template #item="{ element, index }">
          <div class="snap-center shrink-0 first:ml-[calc(50%-160px)] last:mr-[calc(50%-160px)]">
            <div class="w-80 bg-white rounded-xl shadow-xl overflow-hidden transform transition-all duration-300 relative group">
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
                  :src="element.url" 
                  :alt="'Photo ' + element.id" 
                  class="w-full h-56 object-cover"
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
                    Ziehe oder nutze die Pfeile zum Sortieren
                  </p>
                </div>
              </div>
            </div>
          </div>
        </template>
      </draggable>

      <!-- Mobile Instructions -->
      <div class="text-white text-center opacity-75 mt-4">
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
      v-if="photos.length === currentLevelData.requiredPhotos"
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