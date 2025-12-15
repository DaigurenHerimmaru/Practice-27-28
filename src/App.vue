<template>
  <div id="app">
    <!-- Уведомления -->
    <div class="notifications-container" :class="{ 'has-notifications': notifications.length > 0 }">
      <div 
        v-for="notification in notifications" 
        :key="notification.id"
        class="notification"
        :class="notification.type"
      >
        <div class="notification-content">
          <span class="notification-icon">
            <span v-if="notification.type === 'success'">✓</span>
            <span v-else-if="notification.type === 'error'">✗</span>
            <span v-else>ℹ</span>
          </span>
          <div class="notification-text">
            <p class="notification-title">{{ notification.title }}</p>
            <p class="notification-message">{{ notification.message }}</p>
          </div>
          <button @click="removeNotification(notification.id)" class="notification-close">×</button>
        </div>
        <div class="notification-progress" :style="{ width: `${notification.progress}%` }"></div>
      </div>
    </div>

    <main class="main-content" style="display: grid; gap: 20px">
      <div class="color-card" v-if="savedPalettes.length > 0">
        <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 15px;">
          <h3 style="margin: 0;">Сохраненные палитры</h3>
          <button @click="showExportOptions = !showExportOptions" class="export-options-btn">Экспорт</button>
        </div>
        
        <div v-if="showExportOptions" class="export-options">
          <button @click="exportAllPalettes" class="export-btn">Экспорт всех</button>
          <label class="import-btn">
            Импорт
            <input type="file" @change="importPalettes" accept=".json" hidden>
          </label>
          <button @click="copyAllPalettes" class="copy-btn">Копировать все</button>
        </div>

        <div class="saved-palettes">
          <div 
            v-for="(palette, index) in savedPalettes" 
            :key="index"
            class="palette-item"
            @click="loadPalette(palette)"
          >
            <div class="palette-colors">
              <div 
                v-for="(color, colorIndex) in palette.colors" 
                :key="colorIndex"
                class="palette-color"
                :style="{ backgroundColor: color }"
                :title="color"
              ></div>
            </div>
            <div class="palette-info">
              <span>{{ palette.name }}</span>
              <div class="palette-actions-small">
                <button 
                  @click.stop="copyPaletteColors(palette)" 
                  class="copy-palette-btn"
                  title="Копировать цвета"
                >
                🗒️
                </button>
                <button 
                  @click.stop="deletePalette(index)" 
                  class="delete-btn"
                  title="Удалить"
                >
                  ×
                </button>
              </div>
            </div>
          </div>
        </div>
      </div>

      <div class="color-card">
        <div class="card-header">
          <h3>Текущая палитра</h3>
          <div class="palette-actions">
            <input 
              v-model="paletteName" 
              placeholder="Название палитры" 
              class="palette-input"
              @keyup.enter="savePalette"
            />
            <button @click="savePalette" class="save-btn">Сохранить</button>
          </div>
        </div>

        <div class="card-color-bg" :style="{ backgroundColor: baseColor }">
          <div class="card-label">
            <p>{{ baseColor }}</p>
            <input type="color" class="color-picker" v-model="baseColor">
          </div>
        </div>

        <div class="grid">
          <CopyBlock title="HEX:">{{ baseColor.toUpperCase() }}</CopyBlock>
          <CopyBlock title="RGB:">{{ rgbFormated }}</CopyBlock>
          <CopyBlock title="HSV:">{{ hsvFormated }}</CopyBlock>
        </div>

        <div class="shades">
          <CopyColor
            v-for="(shade, index) in shades"
            :key="index"
            :color="shade"
          />
        </div>
      </div>

      <div class="color-card">
        <p>Пара</p>
        <div class="shades" style="justify-content: left;">
          <CopyColor :color="baseColor"/>
          <CopyColor :color="compColor"/>
        </div>
        <p>Тройка</p>
        <div class="shades" style="justify-content: left;">
          <CopyColor
            v-for="(shade, index) in triadColors"
            :key="index"
            :color="shade"
          />
        </div>
        <p>Четвёрка</p>
        <div class="shades" style="justify-content: left;">
          <CopyColor
            v-for="(shade, index) in fourColors"
            :key="index"
            :color="shade"
          />
        </div>
      </div>
    </main>
  </div>
</template>

<script>
import { ref, computed, onMounted } from "vue";
import CopyBlock from "./components/CopyBlock.vue";
import CopyColor from "./components/CopyColor.vue";

export default {
  name: "App",

  components: {
    CopyBlock,
    CopyColor
  },

  setup() {
    const baseColor = ref("#94424f");
    const paletteName = ref("");
    const savedPalettes = ref([]);
    const showExportOptions = ref(false);
    const notifications = ref([]);

    const showNotification = (title, message, type = "success", duration = 3000) => {
      const id = Date.now();
      const notification = {
        id,
        title,
        message,
        type,
        progress: 100,
        duration
      };

      notifications.value.push(notification);

      const interval = setInterval(() => {
        const index = notifications.value.findIndex(n => n.id === id);
        if (index !== -1) {
          notifications.value[index].progress -= 100 / (duration / 50);
          if (notifications.value[index].progress <= 0) {
            removeNotification(id);
            clearInterval(interval);
          }
        }
      }, 50);

      setTimeout(() => {
        removeNotification(id);
      }, duration);
    };

    const removeNotification = (id) => {
      const index = notifications.value.findIndex(n => n.id === id);
      if (index !== -1) {
        notifications.value.splice(index, 1);
      }
    };

    onMounted(() => {
      const saved = localStorage.getItem('colorPalettes');
      if (saved) {
        savedPalettes.value = JSON.parse(saved);
      }
    });

    const saveToLocalStorage = () => {
      localStorage.setItem('colorPalettes', JSON.stringify(savedPalettes.value));
    };

    const rgb = computed({
      get() {
        const hex = baseColor.value
        return { 
          r: parseInt(hex.slice(1, 3), 16), 
          g: parseInt(hex.slice(3, 5), 16), 
          b: parseInt(hex.slice(5, 7), 16) 
        };
      }
    });

    const hsv = computed({
      get() {
        const R = rgb.value.r / 255;
        const G = rgb.value.g / 255;
        const B = rgb.value.b / 255;

        const max = Math.max(R, G, B);
        const min = Math.min(R, G, B);
        const delta = max - min;

        let h = 0;
        if (delta !== 0) {
          if (max === R) h = 60 * (((G - B) / delta) % 6);
          else if (max === G) h = 60 * (((B - R) / delta) + 2);
          else h = 60 * (((R - G) / delta) + 4);
        }
        if (h < 0) h += 360;
        return { 
          h: parseInt(h < 0 ? h+360 : h), 
          s: parseInt((max === 0 ? 0 : delta / max) * 100),
          v: parseInt(max * 100)
        };
      }
    });

    const rgbFormated = computed(() => {
      return `${rgb.value.r}, ${rgb.value.g}, ${rgb.value.b}`;
    });

    const hsvFormated = computed(() => {
      return `${hsv.value.h}, ${hsv.value.s}, ${hsv.value.v}`;
    });

    // HSV -> RGB
    const hsvToRgb = ({ h, s, v }) => {
      const c = v * s;
      const x = c * (1 - Math.abs(((h / 60) % 2) - 1));
      const m = v - c;

      let r1 = 0,
        g1 = 0,
        b1 = 0;
      if (h < 60) [r1, g1, b1] = [c, x, 0];
      else if (h < 120) [r1, g1, b1] = [x, c, 0];
      else if (h < 180) [r1, g1, b1] = [0, c, x];
      else if (h < 240) [r1, g1, b1] = [0, x, c];
      else if (h < 300) [r1, g1, b1] = [x, 0, c];
      else [r1, g1, b1] = [c, 0, x];

      const r = Math.round((r1 + m) * 255);
      const g = Math.round((g1 + m) * 255);
      const b = Math.round((b1 + m) * 255);

      return { r, g, b };
    };

    // RGB -> HEX
    const rgbToHex = ({ r, g, b }) =>
      `#${r.toString(16).padStart(2, "0")}${g
        .toString(16)
        .padStart(2, "0")}${b.toString(16).padStart(2, "0")}`;

    const compColor = computed(() => {
      const {h,s,v} = hsv.value;
      const trgb = hsvToRgb({ h: (h + 180) % 360, s:s/100, v:v/100 });
      return rgbToHex(trgb);
    });

    const triadColors = computed(() => {
      const {h,s,v} = hsv.value;
      return [
        baseColor.value,
        rgbToHex(hsvToRgb({ h: (h + 120) % 360, s:s/100, v:v/100 })),
        rgbToHex(hsvToRgb({ h: (h + 240) % 360, s:s/100, v:v/100 }))
      ];
    });

    const fourColors = computed(() => {
      const {h,s,v} = hsv.value;
      return [
        baseColor.value,
        rgbToHex(hsvToRgb({ h: (h + 90) % 360, s:s/100, v:v/100 })),
        rgbToHex(hsvToRgb({ h: (h + 180) % 360, s:s/100, v:v/100 })),
        rgbToHex(hsvToRgb({ h: (h + 270) % 360, s:s/100, v:v/100 }))
      ];
    });

    const shades = computed(() => {
      const { h, s, v } = hsv.value;

      const shadeCount = 5;
      const result = [];
      const sNormalized = s / 100;
      const vNormalized = v / 100;

      for (let i = 0; i < shadeCount; i++) {
        const newV = vNormalized * (1 - i * 0.15);
        const rgbShade = hsvToRgb({ 
          h, 
          s: sNormalized, 
          v: Math.max(0.05, Math.min(1, newV)) 
        });
        result.push(rgbToHex(rgbShade));
      }
      return result;
    });

    const savePalette = () => {
      if (!paletteName.value.trim()) {
        paletteName.value = `Палитра ${new Date().toLocaleDateString('ru-RU')}`;
      }

      const colors = [
        baseColor.value,
        ...triadColors.value,
        ...fourColors.value,
        ...shades.value
      ];

      const uniqueColors = [...new Set(colors)];

      const newPalette = {
        id: Date.now(),
        name: paletteName.value,
        date: new Date().toLocaleString('ru-RU'),
        colors: uniqueColors,
        baseColor: baseColor.value,
        hsv: hsv.value,
        rgb: rgb.value
      };

      savedPalettes.value.unshift(newPalette);
      saveToLocalStorage();

      showNotification(
        "Палитра сохранена",
        `"${paletteName.value}" добавлена в вашу коллекцию`,
        "success"
      );

      paletteName.value = "";
    };

    const loadPalette = (palette) => {
      baseColor.value = palette.baseColor;
      paletteName.value = palette.name;
    };

    const deletePalette = (index) => {
      const paletteName = savedPalettes.value[index].name;
      savedPalettes.value.splice(index, 1);
      saveToLocalStorage();
    };

    const copyPaletteColors = (palette) => {
      const colorsText = palette.colors.join('\n');
      navigator.clipboard.writeText(colorsText);
      
      showNotification(
        "Цвета скопированы",
        `${palette.colors.length} цветов из "${palette.name}" скопированы`,
        "success"
      );
    };

    const copyAllPalettes = () => {
      const allColors = savedPalettes.value.flatMap(p => p.colors);
      const uniqueColors = [...new Set(allColors)];
      const colorsText = uniqueColors.join('\n');
      
      navigator.clipboard.writeText(colorsText);
      
      showNotification(
        "Все цвета скопированы",
        `${uniqueColors.length} уникальных цветов скопированы`,
        "success"
      );
    };

    const exportAllPalettes = () => {
      const dataStr = JSON.stringify(savedPalettes.value, null, 2);
      const dataUri = 'data:application/json;charset=utf-8,'+ encodeURIComponent(dataStr);
      
      const exportFileDefaultName = `color-palettes-${new Date().toISOString().split('T')[0]}.json`;
      
      const linkElement = document.createElement('a');
      linkElement.setAttribute('href', dataUri);
      linkElement.setAttribute('download', exportFileDefaultName);
      linkElement.click();
      
      showNotification(
        "Палитры экспортированы",
        "Файл сохранен в формате JSON",
        "success"
      );
    };

    const importPalettes = (event) => {
      const file = event.target.files[0];
      if (!file) return;
      
      const reader = new FileReader();
      reader.onload = (e) => {
        try {
          const imported = JSON.parse(e.target.result);
          if (Array.isArray(imported)) {
            savedPalettes.value = imported;
            saveToLocalStorage();
            
            showNotification(
              "Палитры импортированы",
              `Загружено ${imported.length} палитр`,
              "success"
            );
          } else {
            showNotification(
              "Ошибка импорта",
              "Файл содержит некорректные данные",
              "error"
            );
          }
        } catch (error) {
          showNotification(
            "Ошибка импорта",
            "Не удалось прочитать файл",
            "error"
          );
        }
      };
      reader.readAsText(file);
    };

    return { 
      baseColor, 
      rgbFormated, 
      hsvFormated, 
      shades, 
      compColor, 
      triadColors,
      fourColors,
      paletteName,
      savedPalettes,
      showExportOptions,
      notifications,
      savePalette,
      loadPalette,
      deletePalette,
      exportAllPalettes,
      importPalettes,
      copyPaletteColors,
      copyAllPalettes,
      removeNotification
    };
  },
};
</script>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

.grid {
  padding-top: 8px;
  display: grid;
  gap: 5px;
  grid-template-columns: repeat(3, 1fr);
  justify-items: center;
}

body {
  font-family: "Segoe UI", Tahoma, Geneva, Verdana, sans-serif;
  background-color: #171717;
  color: #fff;
  line-height: 1.6;
  padding: 50px;
}

#app {
  display: flex;
  flex-direction: column;
  position: relative;
}

.notifications-container {
  position: fixed;
  top: 20px;
  right: 20px;
  z-index: 1000;
  display: flex;
  flex-direction: column;
  gap: 10px;
  max-width: 350px;
  transition: all 0.3s ease;
}

.notification {
  background: #262626;
  border-radius: 10px;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.3);
  overflow: hidden;
  animation: slideIn 0.3s ease;
  border-left: 4px solid #4CAF50;
  position: relative;
}

.notification.success {
  border-left-color: #4CAF50;
}

.notification.error {
  border-left-color: #f44336;
}

.notification.info {
  border-left-color: #2196F3;
}

.notification-content {
  display: flex;
  align-items: flex-start;
  padding: 15px;
  gap: 12px;
}

.notification-icon {
  font-size: 20px;
  width: 30px;
  height: 30px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.1);
  flex-shrink: 0;
}

.notification.success .notification-icon {
  color: #4CAF50;
}

.notification.error .notification-icon {
  color: #f44336;
}

.notification.info .notification-icon {
  color: #2196F3;
}

.notification-text {
  flex: 1;
  min-width: 0;
}

.notification-title {
  font-weight: 600;
  font-size: 0.95em;
  margin-bottom: 3px;
  color: #fff;
}

.notification-message {
  font-size: 0.85em;
  color: #aaa;
  line-height: 1.4;
}

.notification-close {
  background: none;
  border: none;
  color: #888;
  font-size: 20px;
  cursor: pointer;
  padding: 0;
  width: 24px;
  height: 24px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 4px;
  transition: all 0.2s;
  flex-shrink: 0;
}

.notification-close:hover {
  background: rgba(255, 255, 255, 0.1);
  color: #fff;
}

.notification-progress {
  height: 3px;
  background: rgba(76, 175, 80, 0.7);
  transition: width 0.05s linear;
}

.notification.error .notification-progress {
  background: rgba(244, 67, 54, 0.7);
}

.notification.info .notification-progress {
  background: rgba(33, 150, 243, 0.7);
}

@keyframes slideIn {
  from {
    transform: translateX(100%);
    opacity: 0;
  }
  to {
    transform: translateX(0);
    opacity: 1;
  }
}

@keyframes slideOut {
  from {
    transform: translateX(0);
    opacity: 1;
  }
  to {
    transform: translateX(100%);
    opacity: 0;
  }
}

.color-card {
  width: 450px;
  background: #262626;
  display: block;
  border-radius: 10px;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.1);
  padding: 15px;
  margin-bottom: 20px;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 15px;
}

.card-header h3 {
  margin: 0;
  font-size: 1.2em;
}

.palette-actions {
  display: flex;
  gap: 10px;
  align-items: center;
}

.palette-input {
  padding: 8px 12px;
  border: 1px solid #444;
  border-radius: 6px;
  background: #333;
  color: white;
  font-size: 0.9em;
  width: 150px;
  transition: border-color 0.3s;
}

.palette-input:focus {
  outline: none;
  border-color: #666;
}

.save-btn {
  padding: 8px 16px;
  background: #4CAF50;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.9em;
  transition: all 0.3s;
  display: flex;
  align-items: center;
  gap: 5px;
}

.save-btn:hover {
  background: #45a049;
  transform: translateY(-1px);
}

.export-options-btn {
  padding: 6px 12px;
  background: #2196F3;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.85em;
  transition: all 0.3s;
}

.export-options-btn:hover {
  background: #1976D2;
}

.export-options {
  display: flex;
  gap: 8px;
  margin-bottom: 15px;
  padding: 10px;
  background: rgba(255, 255, 255, 0.05);
  border-radius: 8px;
}

.export-btn, .import-btn, .copy-btn {
  padding: 6px 12px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.85em;
  transition: all 0.3s;
  display: flex;
  align-items: center;
  gap: 5px;
}

.export-btn {
  background: #4CAF50;
  color: white;
}

.import-btn {
  background: #FF9800;
  color: white;
  cursor: pointer;
}

.copy-btn {
  background: #9C27B0;
  color: white;
}

.export-btn:hover, .import-btn:hover, .copy-btn:hover {
  opacity: 0.9;
  transform: translateY(-1px);
}

.card-color-bg {
  display: flex;
  width: 100%;
  height: 150px;
  box-sizing: border-box;
  border-radius: 15px;
  align-content: center;
  align-items: center;
  justify-content: center;
}

.card-label {
  width: 60%;
  background: rgba(38, 38, 38, 0.9);
  display: flex;
  gap: 30px;
  padding: 10px;
  align-items: center;
  justify-content: center;
  border-radius: 10px;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.1);
  backdrop-filter: blur(5px);
}

.card-label p {
  margin: 0;
  max-width: 150px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.color-picker {
  border-radius: 50%;
  background: transparent;
  border: none;
  cursor: pointer;
  height: 40px;
  width: 40px;
  padding: 0;
  vertical-align: middle;
  transition: transform 0.3s;
}

.color-picker:hover {
  transform: scale(1.1);
}

.shades {
  margin-top: 15px;
  display: flex;
  gap: 5px;
  justify-content: center;
  flex-wrap: wrap;
}

/* Стили для сохраненных палитр */
.saved-palettes {
  display: flex;
  flex-direction: column;
  gap: 10px;
  max-height: 300px;
  overflow-y: auto;
  padding-right: 5px;
}

.saved-palettes::-webkit-scrollbar {
  width: 5px;
}

.saved-palettes::-webkit-scrollbar-track {
  background: #333;
  border-radius: 3px;
}

.saved-palettes::-webkit-scrollbar-thumb {
  background: #666;
  border-radius: 3px;
}

.palette-item {
  background: #333;
  border-radius: 8px;
  padding: 12px;
  cursor: pointer;
  transition: all 0.3s;
  border: 1px solid transparent;
}

.palette-item:hover {
  background: #3a3a3a;
  border-color: #444;
  transform: translateY(-2px);
}

.palette-colors {
  display: flex;
  gap: 3px;
  margin-bottom: 10px;
  height: 25px;
}

.palette-color {
  flex: 1;
  border-radius: 3px;
  min-width: 0;
  transition: transform 0.2s;
}

.palette-color:hover {
  transform: scale(1.05);
  z-index: 1;
  position: relative;
}

.palette-info {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 0.9em;
}

.palette-actions-small {
  display: flex;
  gap: 5px;
}

.copy-palette-btn {
  background: #2196F3;
  color: white;
  border: none;
  border-radius: 4px;
  width: 24px;
  height: 24px;
  cursor: pointer;
  font-size: 0.9em;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.3s;
}

.copy-palette-btn:hover {
  background: #1976D2;
}

.delete-btn {
  background: #ff4444;
  color: white;
  border: none;
  border-radius: 50%;
  width: 20px;
  height: 20px;
  cursor: pointer;
  font-size: 1.2em;
  line-height: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.3s;
}

.delete-btn:hover {
  background: #ff0000;
}

@media (max-width: 768px) {
  .main-content {
    grid-template-columns: 1fr;
  }
  
  .color-card {
    width: 100%;
  }
  
  .palette-actions {
    flex-direction: column;
    align-items: stretch;
  }
  
  .palette-input {
    width: 100%;
  }
  
  .notifications-container {
    left: 20px;
    right: 20px;
    max-width: none;
  }
}
</style>