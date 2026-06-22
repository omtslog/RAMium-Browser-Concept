# RAMium: Концепт гибридного веб-браузера нового поколения на базе Chromium

**Автор идеи (Product Owner):** omtslog  
**Дата публикации концепта:** Июнь 2026 года  
**Лицензия:** GNU GPL v3.0  

---

## 🇷🇺 Описание проекта (Русский)

**RAMium** — это концептуальная архитектура веб-браузера на базе движка Chromium, созданная для полного исключения износа SSD-накопителей, ультимативной производительности в играх и абсолютной локальной безопасности. Браузер по умолчанию физически изолирован от постоянной памяти компьютера (SSD/HDD) и переносит всю свою жизнедеятельность в адресное пространство оперативной памяти (ОЗУ) по принципу *Amnesic* (амнезийная система).

### Ключевые архитектурные фичи

#### 1. Нулевой износ SSD по умолчанию (Режим Амнезии)
* **In-Memory сетевой кэш:** Модуль `net::HttpCache` жестко переводится на фабрику `CreateInMemoryBackendFactory()`. Кэш картинок, тяжелых скриптов и видеороликов (включая 4K-стримы) никогда не сбрасывается на диск.
* **Базы данных в ОЗУ:** Все локальные хранилища (история посещений, cookies, веб-хранилища LocalStorage и IndexedDB) инициализируются через SQLite константу `:memory:`. При закрытии процесса сессия бесследно испаряется.
* **Локальная безопасность:** Защита от физического извлечения данных. На диске компьютера не остается заблокированных или «битых» временных файлов.

#### 2. Селективная амнезия, менеджер паролей и «Умный выход»
*   **Изолированный менеджер паролей:** Единственный модуль, имеющий точечный доступ к зашифрованному файлу на SSD через Windows DPAPI. Пароли запрашиваются при старте и сохраняются на диск только по прямому согласию пользователя.
*   **Диалог завершения работы с функцией точечной амнезии:** При нажатии на «крестик» подсистема `BrowserCloseManager` перехватывает событие закрытия и предлагает пользователю гибкий выбор:
    *   `[🗑️ Полная очистка]`: Моментальное уничтожение процессов браузера (`base::Process::Terminate`), ОЗУ полностью освобождается, на SSD не пишется ничего.
    *   `[🎯 Селективное сохранение]`: Вывод кастомного оверлейного UI-окна со списком всех активных дескрипторов вкладок. Пользователь может отметить галочками только нужные страницы (например, рабочий чат или важную статью). Браузер мгновенно уничтожает в памяти 99% фонового кэша, истории и куков невыбранных страниц. Выбранные вкладки и их авторизационные сессии поток-сериализатор собирает из ОЗУ, упаковывает в байтовый поток, шифрует алгоритмом AES-256 и точечно записывает на SSD в один временный файл `session.ram`. При следующем холодном старте RAMium считывает этот файл, разворачивает структуры данных обратно в ОЗУ и немедленно уничтожает файл с диска методом Secure Wipe (запись нулями).

#### 3. Аппаратное управление ресурсами (CPU Affinity для геймеров)
* **Исключение загруженных ядер:** Интеграция системной функции `SetProcessAffinityMask` напрямую в настройки производительности браузера. Возможность вручную или автоматически освобождать первые ядра процессора (CPU 0-3), которые обычно перегружены ОС и тяжелыми играми, для исключения микрофризов (статтеров).
* **Поддержка гибридных процессоров:** Жесткое ограничение работы фоновых вкладок только энергоэффективными ядрами (E-cores для Intel) или малым подмножеством ядер, чтобы браузер не мешал запущенной параллельно игре.

#### 4. Тотальная изоляция загрузок (Встроенная песочница)
* Любой скачиваемый файл (документ, медиа, исполняемый файл) скачивается строго во временный буфер в оперативной памяти.
* Запуск или просмотр файла происходит в изолированном контексте браузера без прямого контакта с файловой системой ОС. Запись на SSD происходит только при явном финальном подтверждении пользователем действия «Сохранить на компьютер».

#### 5. Автоматизированная архитектура обновлений
* Проект реализуется в виде набора автоматизированных патчей (**Git Patches**) и скриптов автосборки. Серверный скрипт отслеживает апдейты официального репозитория Chromium. При выходе обновлений безопасности робот автоматически накладывает патчи RAMium поверх нового кода и компилирует свежий билд.

#### 6. Жесткое аппаратное ограничение GPU (Integrated Graphics Only)
*   **Принудительный перехват графического контекста (DXGI API):**
    При инициализации графического подпроцесса Chromium опрашивает систему. Мы модифицируем файл `gpu/config/gpu_info_collector_win.cc`. Браузер перехватывает список видеокарт через DirectX Graphics Infrastructure API (`IDXGIAdapter::GetDesc`). При включении этого режима RAMium вводит жесткое аппаратное ограничение: дискретная видеокарта (Nvidia/AMD) полностью игнорируется на физическом уровне. Это высвобождает до 1.5–2 ГБ дефицитной видеопамяти (VRAM) и полностью убирает фоновую нагрузку на GPU во время воспроизведения 4K-видео или стримов на втором мониторе.
*   **Бескомпромиссный блок системного переключения:**
    Процесс GPU принудительно запускается со строгими системными флагами `--force-low-power-gpu` и `--gpu-testing-vendor-id/device-id`, указывающими строго на iGPU. Чтобы обойти любые попытки Windows автоматически переключить графику, RAMium отправляет прямой запрос ядру ОС через Windows API функцию `D3DKMTSetProcessSchedulingPriorityClass`, выставляя графическому процессу браузера профиль максимального энергосбережения. Это строго-настрого запрещает системе задействовать ресурсы дискретной видеокарты.

 #### 7. Экосистема In-Memory виртуализации (Мини-ОС, WASM-контейнеры и Игры)
*   **Изолированная WebAssembly-песочница для микро-ОС:**
    Используя высокопроизводительный движок V8 и технологию WebAssembly (WASM), RAMium разворачивает полноценные независимые микро-контейнеры легких операционных систем (например, Alpine Linux или специализированные распределенные ядра) непосредственно в адресном пространстве ОЗУ конкретной вкладки. Виртуальная среда получает эмулируемую файловую систему, динамически аллоцированную внутри кучи (Heap) процесса. Любые вызовы POSIX/Win32 транслируются внутри изолированного контекста памяти, полностью блокируя физические вызовы ввода-вывода (I/O) к реальному дисковому накопителю. Это позволяет разворачивать изолированные среды разработки (Node.js, Python, среды компиляции) в режиме «одноразовой ОС» без риска повреждения хост-системы.
*   **Игровой ретро-комбайн и эмуляция на базе iGPU:**
    Интеграция скомпилированных в WASM ядер эмуляции (архитектуры x86, ARM, а также консольных сред DOS, PlayStation, Sega) с прямой трансляцией графических инструкций через WebGL 2.0 / WebGPU. Все бинарные образы (ROM-файлы), шейдеры и текстуры динамически загружаются в ОЗУ хоста. Архитектура RAMium гарантирует, что эти тяжелые вычислительные процессы жестко заперты внутри встроенного видеоядра (iGPU) и свободных энергоэффективных ядер процессора (E-cores). Это позволяет пользователю параллельно запускать полноценные игровые сессии или тяжелые интерактивные приложения прямо во вкладке браузера (например, на втором мониторе), не отбирая ни единого такта производительности или мегабайта VRAM у основной дискретной видеокарты, на которой запущена AAA-игра. При закрытии вкладки вся аллоцированная под виртуализацию область памяти мгновенно деструктурируется Windows.

#### 8. Интеграция мессенджеров с ОЗУ-изоляцией трафика (Sidebar Sandbox)
*   **Блокировка дискового I/O для веб-чатов:**
    Веб-клиенты мессенджеров (Telegram Web, Discord, WhatsApp) интегрируются в интерфейс браузера через изолированный компонент боковой панели (`Sidebar WebContents`). Их процессы полностью подчиняются глобальной логике RAMium: все медиа-потоки (голосовые сообщения, превью видео, аватарки каналов и тяжелые вложения) обрабатываются исключительно внутри динамического буфера ОЗУ хоста. Физическая запись медиа-кэша на сектор SSD полностью заблокирована.
*   **Амнезийное уничтожение сессий:**
    Токены авторизации и сессионные ключи мессенджеров удерживаются строго в адресном пространстве памяти процесса. При выборе опции «Полная очистка» во время закрытия браузера, токены мгновенно стираются из памяти без возможности восстановления, предотвращая кражу сессий (Session Hijacking) через физический анализ накопителя. При выборе «Селективного сохранения» — токен конкретного мессенджера точечно шифруется алгоритмом AES-256 в контейнер `session.ram` на SSD, разворачиваясь в RAM при следующем запуске и моментально затираясь на диске нулями (Secure Wipe).

#### 9. Встроенный прокси-менеджер и VPN с ОЗУ-изоляцией ключей (Network Isolation)
*   **Низкоуровневая инкапсуляция трафика:**
    Мы модифицируем сетевой слой Chromium (`services/network/`). В сетевой стек интегрируются легковесные клиенты прокси-протоколов (Shadowsocks, VLESS, WireGuard). Сессионные конфигурации и криптографические ключи туннелей генерируются динамически и удерживаются исключительно в адресном пространстве ОЗУ. Физическая запись логов подключения и конфигурационных файлов на SSD полностью исключена. При уничтожении процесса ключи стираются, аннулируя сессию.
*   **Аппаратный Anti-Leak и селективный пинг:**
    Блокировка утечек данных на уровне кода: подсистемы WebRTC и DNS-резолвера принудительно перенаправляются внутрь поднятого прокси-контекста, исключая обходные запросы к серверам провайдера. Реализуется селективная маршрутизация: шифрование применяется строго к выделенным процессам-рендерерам (вкладкам браузера), что гарантирует неизменность прямого сетевого маршрута и минимальный пинг для сторонних процессов ОС (запущенных параллельно сетевых игр).

#### 9.: Децентрализованный P2P-бартер и ОЗУ-изоляция (Value-to-Value Exchange Engine)
*   **Архитектурный баланс обмена мощностей на приватность:**
    *   **RU:** RAMium вводит низкоуровневую систему учета игрового времени в ОЗУ — **Time-for-VIP Ledger**, которая связывает независимых провайдеров (VPN/прокси/пароли) и геймеров в единую экосистему обмена без фиатных денег. Владельцы систем (от 32/64 ГБ ОЗУ) безопасно монетизируют простои железа, пока спят или находятся на работе. Продав, например, 5 часов простоя своего ПК в пул комьюнити для удаленного гейминга на dGPU, пользователь автоматически активирует Pro-режим браузера. RAMium выступает прозрачным регулятором, переводя ценность от аренды сторонним разработчикам софта в качестве честной платы за их инфраструктуру. Покупатели со слабых ПК, приобретая, например, пакет из 10 часов игрового времени у хостов, также автоматически поощряются бесплатным VIP-статусом в браузере.
*   **Двусторонняя криптографическая защита в ОЗУ:**
    *   **RU:** Безопасность в экосистеме является абсолютно симметричной и обеспечивается изоляцией на уровне памяти. Удаленный гостевой процесс разворачивается СТРОГО внутри изолированной ОЗУ-песочницы (In-Memory Isolation Sandbox), полностью исключая физический доступ к файловой системе (SSD) и учетным записям владельца ПК. С другой стороны, вводимые гостем пароли от его игровых аккаунтов (Steam, Epic Games) и игровой кэш обрабатываются исключительно в выделенных блоках памяти хоста, к которым у владельца ПК нет доступа. При завершении сессии вся выделенная область памяти хоста мгновенно уничтожается методом Secure Wipe (запись нулями) — сессия гостя бесследно испаряется.





---

## 🇺🇸 Project Description (English)

**RAMium** is a conceptual web browser architecture based on the Chromium engine, designed to completely eliminate SSD wear, deliver ultimate gaming performance, and ensure absolute local security. By default, the browser is physically isolated from non-volatile storage (SSD/HDD) and operates entirely within the RAM address space, following the *Amnesic* system principle.

### Key Architectural Features

#### 1. Zero SSD Wear by Default (Amnesia Mode)
* **In-Memory Network Cache:** The `net::HttpCache` module is forced to use the `CreateInMemoryBackendFactory()` factory. Images, heavy scripts, and video streams (including 4K) are never cached to the disk.
* **RAM-Based Databases:** All local storage (browsing history, cookies, LocalStorage, and IndexedDB) is initialized using the SQLite `:memory:` constant. Once the process terminates, the session vanishes without a trace.
* **Local Security:** Protection against physical data extraction. No locked or corrupted temporary files remain on the drive.

#### 2. Selective Amnesia, Password Manager & "Smart Close"
*   **Isolated Password Manager:** The only module allowed pinpoint access to an encrypted file on the SSD via Windows DPAPI. Passwords are read at startup and written to the disk only with explicit user consent.
*   **Session Termination Dialog with Pinpoint Amnesia:** Upon clicking the close button, the `BrowserCloseManager` subsystem intercepts the event and prompts a flexible choice:
    *   `[🗑️ Wipe Everything]`: Immediate process termination (`base::Process::Terminate`) and complete RAM clearance; absolutely zero data is written to the SSD.
    *   `[🎯 Selective Preservation]`: Triggers a custom overlay UI listing all active tab handles. The user can toggle checkboxes to save only specific pages (e.g., a work chat or an important article). The browser instantly purges 99% of background cache, history, and cookies of unselected pages from memory. A serialization thread marshals only the chosen tabs and their auth sessions from RAM into a byte stream, encrypts it via AES-256, and writes a single temporary `session.ram` file to the SSD. On the subsequent cold boot, RAMium reads this file, expands the structures back into RAM, and immediately overwrites the file on the disk with zeroes (Secure Wipe).

#### 3. Hardware Resource Management (CPU Affinity for Gamers)
* **Core Exclusion:** Integrating the native `SetProcessAffinityMask` API directly into the browser's performance settings. Users can manually or automatically free up the first CPU cores (e.g., CPU 0-3)—which are usually overloaded by the OS and games—to eliminate micro-stutters.
* **Hybrid Processor Support:** Explicitly confining background processes to energy-efficient cores (Intel's E-cores) or a small subset of cores, ensuring the browser never interferes with a running game.

#### 4. Total Download Isolation (Built-in Sandbox)
* Every downloaded file (documents, media, executables) is saved strictly into a temporary buffer within RAM.
* Viewing or running files occurs in an isolated browser context with zero file system access. Writing to the SSD occurs only when the user explicitly confirms the "Save to Disk" action.

#### 5. Automated Update Architecture
* The project is maintained as a set of automated **Git Patches** and build scripts. A server-side script tracks updates upstream in the official Chromium repository. When security patches drop, the automation applies RAMium patches onto the new code and compiles a fresh build.

#### 6. Hard Hardware Restriction (Integrated Graphics Only)
*   **Enforced Graphics Context Interception (DXGI API):**
    During GPU sub-process initialization, Chromium queries the OS. We modify `gpu/config/gpu_info_collector_win.cc`. The browser intercepts the display adapter list via the DirectX Graphics Infrastructure API (`IDXGIAdapter::GetDesc`). When active, RAMium enforces a hard hardware restriction: the discrete graphics card (Nvidia/AMD) is completely ignored at the physical level. This liberates up to 1.5–2 GB of precious VRAM and eliminates background GPU overhead during 4K video or stream playback on secondary displays.
*   **Uncompromising Offloading Lock:**
    The GPU process is forced to initialize with strict system flags, including `--force-low-power-gpu` and targeted `--gpu-testing-vendor-id/device-id` variables pointing directly to the iGPU. To tightly enforce this restriction against Windows-level overrides, RAMium sends a direct request to the OS kernel via the Windows native `D3DKMTSetProcessSchedulingPriorityClass` API, setting the browser's GPU thread priority profile to ultra power-saving mode, completely locking out discrete GPU resources.

#### 7. In-Memory Virtualization Ecosystem (Mini-OS, WASM-Containers & Embedded Gaming)
*   **Isolated WebAssembly Sandbox for Micro-OS:**
    Leveraging the high-performance V8 engine and WebAssembly (WASM) compiler tech, RAMium spawns self-contained, independent micro-containers of lightweight operating systems (e.g., Alpine Linux or tailored modular kernels) directly within the RAM address space of a specific tab process. The virtual stack is backed by an emulated file system dynamically allocated inside the process heap. All POSIX/Win32 system calls are translated entirely within this volatile sandbox, executing absolute I/O blocking against the physical host drive. This structure enables safe execution of disposable development environments (Node.js, Python, compiler toolchains) acting as single-use OS instances with zero malware persistence risk.
*   **Embedded iGPU-Driven Retro Gaming & Emulation:**
    Direct integration of WASM-compiled emulation cores (x86, ARM, and vintage console runtimes like DOS, PlayStation, Sega) utilizing native WebGL 2.0 / WebGPU pipelining. All binary disk images (ROMs), shaders, and textures are dynamically allocated into the host system RAM. RAMium's architecture guarantees these execution pipelines are strictly bound to the integrated graphics core (iGPU) and efficiency cores (E-cores). Users can run distinct gaming sessions or heavy interactive apps right inside a tab (e.g., on a secondary display) without stripping a single clock cycle or a megabyte of VRAM from the primary discrete GPU running a heavy AAA-title. Closing the tab forces Windows to immediately release and reclaim the entire virtualization memory sub-block.

#### 8. Messenger Integration with RAM-Isolated Traffic (Sidebar Sandbox)
*   **Disk I/O Suppression for Web Chats:**
    Web clients (Telegram Web, Discord, WhatsApp) are embedded into the UI via a dedicated, isolated sidebar component (`Sidebar WebContents`). Their sub-processes comply entirely with RAMium’s global core logic: all media streams (voice notes, video previews, channel avatars, and heavy attachments) are processed strictly inside dynamic host RAM buffers. Physical media caching to SSD sectors is completely blocked.
*   **Amnesic Session Destruction:**
    Authentication tokens and session keys are held exclusively within the volatile process memory space. If "Wipe Everything" is selected on browser exit, these tokens are instantly purged from RAM with zero recovery footprint, preventing session hijacking through post-mortem hardware drive forensics. If "Selective Preservation" is toggled, the specific messenger's token is pinpoint-encrypted via AES-256 into the `session.ram` container on the SSD, expanding back into RAM on the next boot and immediately undergoing a secure zero-fill overwrite (Secure Wipe) on the disk.

#### 9. Built-in Proxy Manager & VPN with RAM-Isolated Keys (Network Isolation)
*   **Low-Level Traffic Encapsulation:**
    We modify Chromium’s network layer (`services/network/`) to embed lightweight proxy clients (Shadowsocks, VLESS, WireGuard) straight into the network stack. Session configurations and cryptographic tunnel keys are generated dynamically and held strictly within the volatile RAM space. Writing connection logs or config files to the SSD is completely blocked. When the process terminates, the keys are instantly purged, rendering past traffic captures unrecoverable.
*   **Hardware Anti-Leak & Selective Routing:**
    Enforcing data leak prevention at the code level: the WebRTC and DNS resolver subsystems are forced into the proxy context, eliminating bypass leaks to ISP servers. Selective routing is implemented to ensure encryption applies exclusively to individual renderer processes (browser tabs), keeping the direct network route and minimum ping fully intact for external OS applications like running multiplayer games.

#### 9. Decentralized P2P Barter & In-Memory Isolation (Value-to-Value Exchange Engine)
*   **Architectural Symmetry of Hardware-to-Privacy Barter:**
    *   **ENG:** RAMium introduces a low-level in-memory billing engine—**Time-for-VIP Ledger**, linking independent security vendors (VPN/proxies/vaults) and gamers into a fiat-free value exchange ecosystem. Systems owners (32/64 GB RAM or higher) safely monetize hardware downtime while they sleep or are away at work. By sharing, for example, 5 hours of host dGPU runtime to the community pool, the user automatically triggers the browser's Pro status. RAMium acts as an automated clearinghouse, seamlessy shifting value from hardware runtime to compensate third-party software developers for their premium infrastructure. Laptop clients purchasing a 10-hour pack of streaming runtime are also instantly awarded a complimentary VIP status on their browser instance.
*   **Two-Way Cryptographic Protection in RAM:**
    *   **ENG:** Ecosystem security is entirely symmetrical and heavily driven by memory isolation profiles. The remote guest stream runs on the dGPU but deploys STRICTLY inside an isolated, volatile RAM container (In-Memory Isolation Sandbox), completely blocking physical I/O calls to the host's operating system or SSD storage. Conversely, the guest's inputs, entered game account credentials (Steam, Epic Games), and live memory cache are handled strictly within hidden allocated RAM sub-blocks on the host, invisible to the hardware provider. The moment the session terminates, the allocated RAM underwent an immediate Secure Wipe (zero-fill overwrite)—leaving zero recovery footprint and vanishing without a trace.







---

## 🛠️ Status / Статус
The project is currently in the **Concept / RFC (Request for Comments)** stage. We are looking for C++ developers and Chromium Embedded Framework (CEF) engineers to build the first prototype.

Проект находится на стадии архитектурного проектирования. Открыт для предложений C++ разработчиков и инженеров.

## 🤝 How to Contribute / Как помочь проекту

### 🇷🇺 Для русскоязычных разработчиков
Привет! Я — **omtslog**, автор идеи и продукт-менеджер проекта RAMium. Я не являюсь C++ программистом, но я детально продумал архитектуру, Win32 API механизмы и логику этого браузера, так как глубоко понимаю боли геймеров и продвинутых пользователей. 

Если вы C++ инженер, специалист по Chromium Embedded Framework (CEF) или системный архитектор — этот проект открыт для вас! Вы можете помочь проекту следующими способами:
1.  **Обсуждение архитектуры (RFC):** Зайдите во вкладку **Issues** и предложите свои идеи по улучшению логики работы кэша или алгоритма CPU Affinity.
2.  **Разработка кода:** Сделайте **Fork** этого репозитория, начните реализацию любого из Этапов (Roadmap) и отправьте мне **Pull Request**. Я с радостью изучу ваш код и объединю его с основным проектом.
3.  **Продвижение:** Поставьте этому проекту ⭐ **Star** на GitHub, чтобы о концепте RAMium узнало как можно больше людей.

---

### 🇺🇸 For International Developers
Hi there! I am **omtslog**, the Product Owner and vision holder of RAMium. While I am not a C++ developer myself, I have meticulously designed the architectural logic, Win32 API integration sequences, and features based on real-world gamer and power-user bottlenecks.

If you are a C++ engineer, a Chromium Embedded Framework (CEF) expert, or a systems architect—this project needs you! Here is how you can pitch in:
1.  **Architecture Review (RFC):** Head over to the **Issues** tab to discuss, critique, or optimize our memory caching and CPU core allocation logic.
2.  **Code Contributions:** Feel free to **Fork** this repository, start hacking on any of the Roadmap phases, and submit a **Pull Request**. I will gladly review and merge your code into the main tree.
