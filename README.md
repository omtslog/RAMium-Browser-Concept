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

* #### 6: Жесткое аппаратное ограничение GPU (Integrated Graphics Only)
*   **Принудительный перехват графического контекста (DXGI API):**
    При инициализации графического подпроцесса Chromium опрашивает систему. Мы модифицируем файл `gpu/config/gpu_info_collector_win.cc`. Браузер перехватывает список видеокарт через DirectX Graphics Infrastructure API (`IDXGIAdapter::GetDesc`). При включении этого режима RAMium вводит жесткое аппаратное ограничение: дискретная видеокарта (Nvidia/AMD) полностью игнорируется на физическом уровне. Это высвобождает до 1.5–2 ГБ дефицитной видеопамяти (VRAM) и полностью убирает фоновую нагрузку на GPU во время воспроизведения 4K-видео или стримов на втором мониторе.
*   **Бескомпромиссный блок системного переключения:**
    Процесс GPU принудительно запускается со строгими системными флагами `--force-low-power-gpu` и `--gpu-testing-vendor-id/device-id`, указывающими строго на iGPU. Чтобы обойти любые попытки Windows автоматически переключить графику, RAMium отправляет прямой запрос ядру ОС через Windows API функцию `D3DKMTSetProcessSchedulingPriorityClass`, выставляя графическому процессу браузера профиль максимального энергосбережения. Это строго-настрого запрещает системе задействовать ресурсы дискретной видеокарты.


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

* ###### Phase 6: Hard Hardware Restriction (Integrated Graphics Only)
*   **Enforced Graphics Context Interception (DXGI API):**
    During GPU sub-process initialization, Chromium queries the OS. We modify `gpu/config/gpu_info_collector_win.cc`. The browser intercepts the display adapter list via the DirectX Graphics Infrastructure API (`IDXGIAdapter::GetDesc`). When active, RAMium enforces a hard hardware restriction: the discrete graphics card (Nvidia/AMD) is completely ignored at the physical level. This liberates up to 1.5–2 GB of precious VRAM and eliminates background GPU overhead during 4K video or stream playback on secondary displays.
*   **Uncompromising Offloading Lock:**
    The GPU process is forced to initialize with strict system flags, including `--force-low-power-gpu` and targeted `--gpu-testing-vendor-id/device-id` variables pointing directly to the iGPU. To tightly enforce this restriction against Windows-level overrides, RAMium sends a direct request to the OS kernel via the Windows native `D3DKMTSetProcessSchedulingPriorityClass` API, setting the browser's GPU thread priority profile to ultra power-saving mode, completely locking out discrete GPU resources.



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
