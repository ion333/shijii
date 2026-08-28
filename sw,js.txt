// 食迹 Service Worker - 离线缓存
const CACHE_NAME = 'shiji-cache-v2.3.0';
const CORE_ASSETS = [
    './index.html',
    './manifest.json'
];

// 安装：缓存核心文件
self.addEventListener('install', (event) => {
    event.waitUntil(
        caches.open(CACHE_NAME)
            .then((cache) => cache.addAll(CORE_ASSETS))
            .then(() => self.skipWaiting())
    );
});

// 激活：清理旧缓存
self.addEventListener('activate', (event) => {
    event.waitUntil(
        caches.keys().then((cacheNames) => {
            return Promise.all(
                cacheNames
                    .filter((name) => name !== CACHE_NAME)
                    .map((name) => caches.delete(name))
            );
        }).then(() => self.clients.claim())
    );
});

// 拦截请求：缓存优先，网络回退
self.addEventListener('fetch', (event) => {
    // 只缓存GET请求
    if (event.request.method !== 'GET') return;

    event.respondWith(
        caches.match(event.request)
            .then((cachedResponse) => {
                // 缓存命中，返回缓存
                if (cachedResponse) {
                    // 后台更新缓存
                    fetch(event.request).then((networkResponse) => {
                        if (networkResponse && networkResponse.status === 200) {
                            caches.open(CACHE_NAME).then((cache) => {
                                cache.put(event.request, networkResponse.clone());
                            });
                        }
                    }).catch(() => {});
                    return cachedResponse;
                }

                // 缓存未命中，从网络获取
                return fetch(event.request).then((networkResponse) => {
                    // 缓存有效响应
                    if (networkResponse && networkResponse.status === 200 && networkResponse.type === 'basic') {
                        const responseToCache = networkResponse.clone();
                        caches.open(CACHE_NAME).then((cache) => {
                            cache.put(event.request, responseToCache);
                        });
                    }
                    return networkResponse;
                }).catch(() => {
                    // 网络失败，返回离线页面
                    return caches.match('./index.html');
                });
            })
    );
});
