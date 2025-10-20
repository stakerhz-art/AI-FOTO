// AI-Image-Generator.jsx
// Красивый одностраничный React-компонент с TailwindCSS, который позволяет пользователю вводить запрос
// и генерировать изображения через бекендный API (/api/generate).
// Инструкция: подключите этот компонент в своё приложение (Next.js / Vite + React).
// Реализуйте на сервере endpoint /api/generate, который принимает JSON { prompt, style, size }
// и возвращает массив URL-адресов сгенерированных изображений (или base64). Ниже — UI и логика.

import React, { useState, useRef } from "react";

export default function AIImageGenerator() {
  const [prompt, setPrompt] = useState("");
  const [style, setStyle] = useState("Фотореализм");
  const [size, setSize] = useState("1024x1024");
  const [images, setImages] = useState([]); // [{id, url, prompt, createdAt}]
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [num, setNum] = useState(3);
  const controllerRef = useRef(null);

  async function handleGenerate(e) {
    e?.preventDefault();
    if (!prompt.trim()) {
      setError("Пожалуйста, напишите запрос для генерации изображения.");
      return;
    }
    setError(null);
    setLoading(true);

    // Отменяем предыдущий запрос, если есть
    if (controllerRef.current) controllerRef.current.abort();
    controllerRef.current = new AbortController();

    try {
      const payload = { prompt, style, size, num };
      const res = await fetch("/api/generate", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(payload),
        signal: controllerRef.current.signal,
      });

      if (!res.ok) {
        const text = await res.text();
        throw new Error(`Ошибка сервера: ${res.status} ${text}`);
      }

      const data = await res.json();
      // Ожидаем: { images: [ { url } ] } или { images: [ { id, url } ] }
      if (!Array.isArray(data.images)) throw new Error("Неверный формат ответа от API");

      const now = new Date().toISOString();
      const newImages = data.images.map((it, idx) => ({
        id: it.id ?? `${Date.now()}-${idx}`,
        url: it.url ?? it, // если сервер возвращает просто строки
        prompt,
        style,
        size,
        createdAt: now,
      }));

      // добавляем новые изображения в начало
      setImages(prev => [...newImages, ...prev].slice(0, 100)); // храним максимум 100
    } catch (err) {
      if (err.name === 'AbortError') {
        setError('Запрос отменён');
      } else {
        console.error(err);
        setError(err.message || 'Ошибка при генерации');
      }
    } finally {
      setLoading(false);
    }
  }

  function handleClear() {
    setImages([]);
  }

  function handleDownload(url, filename = 'ai-image.png') {
    // Классическая загрузка через временную ссылку
    const a = document.createElement('a');
    a.href = url;
    a.download = filename;
    document.body.appendChild(a);
    a.click();
    a.remove();
  }

  return (
    <div className="min-h-screen bg-gradient-to-b from-gray-50 to-white p-8">
      <div className="max-w-6xl mx-auto">
        {/* Header */}
        <header className="flex items-center justify-between mb-8">
          <div>
            <h1 className="text-3xl font-extrabold tracking-tight mb-1">Генератор AI-изображений</h1>
            <p className="text-sm text-gray-600">Вводите запрос — и получайте красивые изображения, созданные ИИ.</p>
          </div>
          <div className="text-sm text-gray-500">Интерфейс на Tailwind • Подключите свой backend</div>
        </header>

        {/* Controls & Form */}
        <form onSubmit={handleGenerate} className="bg-white shadow rounded-lg p-6 mb-6">
          <div className="grid grid-cols-1 md:grid-cols-4 gap-4">
            <div className="md:col-span-3">
              <label className="block text-sm font-medium text-gray-700 mb-2">Запрос (prompt)</label>
              <input
                value={prompt}
                onChange={(e) => setPrompt(e.target.value)}
                placeholder={`Например: "Портрет девушки в стиле киберпанк, неоновое освещение, детализированная композиция"`}
                className="w-full rounded-md border-gray-200 shadow-sm focus:ring-2 focus:ring-indigo-200 p-3"
              />
              <div className="flex items-center gap-3 mt-2 text-xs text-gray-500">
                <span>Советы: добавьте стиль, эмоции, освещение, цветовую палитру.</span>
              </div>
            </div>

            <div className="space-y-2">
              <label className="block text-sm font-medium text-gray-700">Стиль</label>
              <select value={style} onChange={(e) => setStyle(e.target.value)} className="w-full p-2 rounded-md border-gray-200">
                <option>Фотореализм</option>
                <option>Иллюстрация</option>
                <option>Вектор</option>
                <option>Акварель</option>
                <option>Киберпанк</option>
                <option>Ретрo</option>
              </select>

              <label className="block text-sm font-medium text-gray-700 mt-2">Размер</label>
              <select value={size} onChange={(e) => setSize(e.target.value)} className="w-full p-2 rounded-md border-gray-200">
                <option>512x512</option>
                <option>768x768</option>
                <option>1024x1024</option>
                <option>1536x1536</option>
              </select>
            </div>
          </div>

          <div className="flex items-center gap-3 mt-4">
            <label className="text-sm text-gray-600">Количество:</label>
            <input type="number" min={1} max={10} value={num} onChange={e => setNum(Number(e.target.value))} className="w-20 p-2 rounded-md border-gray-200" />

            <button type="submit" disabled={loading} className="ml-auto inline-flex items-center gap-2 bg-indigo-600 text-white px-4 py-2 rounded-md shadow hover:opacity-95 disabled:opacity-60">
              {loading ? 'Генерируем...' : 'Сгенерировать'}
            </button>

            <button type="button" onClick={() => { setPrompt(''); setStyle('Фотореализм'); setSize('1024x1024'); }} className="px-3 py-2 border rounded-md text-sm">Сбросить поля</button>

            <button type="button" onClick={handleClear} className="px-3 py-2 border rounded-md text-sm">Очистить результаты</button>
          </div>

          {error && <div className="mt-4 text-sm text-red-600">{error}</div>}
        </form>

        {/* Results */}
        <section>
          <div className="flex items-center justify-between mb-4">
            <h2 className="text-xl font-semibold">Результаты</h2>
            <div className="text-sm text-gray-500">Всего: {images.length}</div>
          </div>

          {images.length === 0 ? (
            <div className="bg-white border border-dashed border-gray-200 rounded-lg p-8 text-center text-gray-400">Здесь появятся сгенерированные изображения</div>
          ) : (
            <div className="grid grid-cols-1 sm:grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-4">
              {images.map(img => (
                <div key={img.id} className="bg-white rounded-lg shadow overflow-hidden">
                  <div className="relative pb-[100%]">{/* квадратная карточка (обёртка) */}
                    <img src={img.url} alt={img.prompt} className="absolute inset-0 w-full h-full object-cover" />
                  </div>
                  <div className="p-3">
                    <div className="text-xs text-gray-600 mb-2 truncate">{img.prompt}</div>
                    <div className="flex items-center gap-2">
                      <button onClick={() => handleDownload(img.url, `ai-${img.id}.png`)} className="text-sm px-3 py-1 border rounded">Скачать</button>
                      <button onClick={() => navigator.clipboard?.writeText(img.url)} className="text-sm px-3 py-1 border rounded">Копировать ссылку</button>
                      <button onClick={() => setImages(prev => prev.filter(i => i.id !== img.id))} className="text-sm px-3 py-1 border rounded">Удалить</button>
                    </div>
                  </div>
                </div>
              ))}
            </div>
          )}
        </section>

        {/* Footer / Tips */}
        <footer className="mt-8 text-sm text-gray-500">
          <p>Серверная часть должна обращаться к выбранной модельной API (OpenAI, Stability, Midjourney, DALLE и т.д.).</p>
          <p className="mt-2">Рекомендация: валидируйте запросы, лимитируйте частоту и подавайте изображения через CDN для производительности.</p>
        </footer>
      </div>
    </div>
  );
}
