Ниже запуск Ollama в Podman, докачка моделей, проблемы с `context deadline exceeded`, восстановление manifest, проверка моделей и работа через контейнер.  

---

# 🐳 Запуск Ollama в Podman и установка моделей (CPU / rootless)

Этот проект описывает, как корректно запускать **Ollama внутри Podman (rootless)**, скачивать большие модели (Qwen, Gemma, Llama, Stable‑Code) и решать типичные проблемы вроде:

- `context deadline exceeded`
- зависание на `pulling manifest`
- повреждённый manifest
- частично скачанные модели

README подходит для Fedora, Arch, Ubuntu и любых систем с Podman.

---

## 📦 Запуск Ollama в Podman

Контейнер запускается в rootless‑режиме и использует сеть хоста, чтобы API было доступно по:

```
http://127.0.0.1:11434
```

### Создание контейнера

```bash
podman run -d \
  --name ollama \
  --network host \
  -v ollama:/root/.ollama \
  docker.io/ollama/ollama:latest
```

Проверка:

```bash
curl http://127.0.0.1:11434/api/tags
```

---

## 📥 Установка моделей внутри контейнера

Все команды выполняются через `podman exec`:

```bash
podman exec -it ollama ollama pull qwen3:8b
podman exec -it ollama ollama pull gemma3:4b
```

---

## ⚠️ Типичная проблема: `context deadline exceeded`

При скачивании больших моделей Podman может оборвать соединение:

```
Error: context deadline exceeded
```

Это **не ошибка модели** — это таймаут rootless Podman.

### ✔ Решение: повторить команду

Ollama докачивает слои:

```bash
podman exec -it ollama ollama pull qwen3:8b
```

### ✔ Ограничить параллельные загрузки (лучший способ)

```bash
podman exec -it ollama env OLLAMA_MAX_PARALLEL_DOWNLOADS=1 ollama pull qwen3:8b
```

### ✔ Перезапуск контейнера с увеличенными таймаутами

```bash
podman stop ollama
podman rm ollama

podman run -d \
  --name ollama \
  --network host \
  -v ollama:/root/.ollama \
  -e OLLAMA_MAX_PARALLEL_DOWNLOADS=1 \
  -e OLLAMA_KEEP_ALIVE=300 \
  docker.io/ollama/ollama:latest
```

---

## 🧩 Проблема: бесконечное `pulling manifest`

Если скачивание оборвалось, manifest может повредиться.  
Слои уже скачаны, но Ollama не может собрать модель.

### ✔ Проверка наличия файлов модели

```bash
podman exec -it ollama ls -lh /root/.ollama/models
```

Если каталог модели есть — слои скачаны.

### ✔ Удаление повреждённого manifest

```bash
podman exec -it ollama rm -f /root/.ollama/models/manifests/gemma3:4b
podman exec -it ollama rm -f /root/.ollama/models/manifests/qwen3:8b
```

После этого:

```bash
podman exec -it ollama ollama pull gemma3:4b
```

Докачка займёт секунды — слои уже на месте.

---

## 🔍 Проверка установленных моделей

### Список моделей

```bash
podman exec -it ollama ollama list
```

### Проверка API

```bash
curl http://127.0.0.1:11434/api/tags
```

### Тестовый запуск модели

```bash
podman exec -it ollama ollama run qwen3:8b "Привет"
```

---

## 🗑 Удаление модели

```bash
podman exec -it ollama ollama rm qwen3:8b
```

---

## 🛑 Полное удаление Ollama из Podman

```bash
podman stop ollama
podman rm ollama
podman volume rm ollama
```

---

## 📌 Итоги

- Ollama отлично работает в Podman, но rootless‑режим требует настройки.
- Большие модели могут рвать соединение → используйте `OLLAMA_MAX_PARALLEL_DOWNLOADS=1`.
- Если зависает на `pulling manifest` → удалите manifest, не трогая слои.
- Проверяйте модели через `ollama list` и API.

---

Если хочешь:

- **GPU‑ускорение (Intel/AMD iGPU через Vulkan)**
- **docker-compose / podman-compose**
- **интеграция с VS Code (Llama‑Coder)**
- **как использовать модели из хоста внутри Podman**


