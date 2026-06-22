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

#### 2. Селективное сохранение и «Умный выход»
* **Изолированный менеджер паролей:** Единственный модуль, имеющий точечный доступ к зашифрованному файлу на SSD через Windows DPAPI. Пароли запрашиваются при старте и сохраняются на диск только по прямому согласию пользователя.
* **Диалог завершения работы:** При нажатии на «крестик» подсистема `BrowserCloseManager` перехватывает событие и предлагает выбор:
  * *[🗑️ Стереть всё]:* Моментальное уничтожение процесса и очистка ОЗУ.
  * *[💾 Сохранить сессию]:* Упаковка текущих вкладок и куков в один зашифрованный временный файл на SSD. При следующем старте файл разворачивается в ОЗУ и тут же стирается с диска.

#### 3. Аппаратное управление ресурсами (CPU Affinity для геймеров)
* **Исключение загруженных ядер:** Интеграция системной функции `SetProcessAffinityMask` напрямую в настройки производительности браузера. Возможность вручную или автоматически освобождать первые ядра процессора (CPU 0-3), которые обычно перегружены ОС и тяжелыми играми, для исключения микрофризов (статтеров).
* **Поддержка гибридных процессоров:** Жесткое ограничение работы фоновых вкладок только энергоэффективными ядрами (E-cores для Intel) или малым подмножеством ядер, чтобы браузер не мешал запущенной параллельно игре.

#### 4. Тотальная изоляция загрузок (Встроенная песочница)
* Любой скачиваемый файл (документ, медиа, исполняемый файл) скачивается строго во временный буфер в оперативной памяти.
* Запуск или просмотр файла происходит в изолированном контексте браузера без прямого контакта с файловой системой ОС. Запись на SSD происходит только при явном финальном подтверждении пользователем действия «Сохранить на компьютер».

#### 5. Автоматизированная архитектура обновлений
* Проект реализуется в виде набора автоматизированных патчей (**Git Patches**) и скриптов автосборки. Серверный скрипт отслеживает апдейты официального репозитория Chromium. При выходе обновлений безопасности робот автоматически накладывает патчи RAMium поверх нового кода и компилирует свежий билд.

---

## 🇺🇸 Project Description (English)

**RAMium** is a conceptual web browser architecture based on the Chromium engine, designed to completely eliminate SSD wear, deliver ultimate gaming performance, and ensure absolute local security. By default, the browser is physically isolated from non-volatile storage (SSD/HDD) and operates entirely within the RAM address space, following the *Amnesic* system principle.

### Key Architectural Features

#### 1. Zero SSD Wear by Default (Amnesia Mode)
* **In-Memory Network Cache:** The `net::HttpCache` module is forced to use the `CreateInMemoryBackendFactory()` factory. Images, heavy scripts, and video streams (including 4K) are never cached to the disk.
* **RAM-Based Databases:** All local storage (browsing history, cookies, LocalStorage, and IndexedDB) is initialized using the SQLite `:memory:` constant. Once the process terminates, the session vanishes without a trace.
* **Local Security:** Protection against physical data extraction. No locked or corrupted temporary files remain on the drive.

#### 2. Selective Storage & "Smart Close"
* **Isolated Password Manager:** The only module allowed pinpoint access to an encrypted file on the SSD via Windows DPAPI. Passwords are read at startup and written to the disk only with explicit user consent.
* **Session Termination Dialog:** Upon clicking the close button, the `BrowserCloseManager` subsystem intercepts the event and prompts a choice:
  * *[🗑️ Wipe Everything]:* Immediate process termination and RAM clearance.
  * *[💾 Save Session]:* Compressing current tabs and cookies into a single encrypted temporary file on the SSD. On the next launch, this file expands into RAM and is instantly deleted from the disk.

#### 3. Hardware Resource Management (CPU Affinity for Gamers)
* **Core Exclusion:** Integrating the native `SetProcessAffinityMask` API directly into the browser's performance settings. Users can manually or automatically free up the first CPU cores (e.g., CPU 0-3)—which are usually overloaded by the OS and games—to eliminate micro-stutters.
* **Hybrid Processor Support:** Explicitly confining background processes to energy-efficient cores (Intel's E-cores) or a small subset of cores, ensuring the browser never interferes with a running game.

#### 4. Total Download Isolation (Built-in Sandbox)
* Every downloaded file (documents, media, executables) is saved strictly into a temporary buffer within RAM.
* Viewing or running files occurs in an isolated browser context with zero file system access. Writing to the SSD occurs only when the user explicitly confirms the "Save to Disk" action.

#### 5. Automated Update Architecture
* The project is maintained as a set of automated **Git Patches** and build scripts. A server-side script tracks updates upstream in the official Chromium repository. When security patches drop, the automation applies RAMium patches onto the new code and compiles a fresh build.

---

## 🛠️ Status / Статус
The project is currently in the **Concept / RFC (Request for Comments)** stage. We are looking for C++ developers and Chromium Embedded Framework (CEF) engineers to build the first prototype.

Проект находится на стадии архитектурного проектирования. Открыт для предложений C++ разработчиков и инженеров.
