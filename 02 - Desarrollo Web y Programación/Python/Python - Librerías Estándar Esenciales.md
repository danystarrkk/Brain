---
title: "Python - Librerías Estándar Esenciales"
tags:
  - lang/python
  - concept/stdlib
  - concept/utilities
  - level/intermedio
  - type/reference
  - area/scripting
created: 2024-01-15
updated: 2024-09-25
status: active
---

# Python - Librerías Estándar Esenciales

> **Objetivo**: Referencia rápida de los módulos built-in más usados. "Baterías incluidas".

---

## 1. Sistema y Entorno

| Módulo | Uso Principal | Funciones Clave |
|--------|---------------|-----------------|
| `os` | Interacción SO | `getcwd()`, `listdir()`, `mkdir()`, `environ`, `path.join()`, `walk()` |
| `sys` | Intérprete | `argv`, `path`, `stdin/stdout/stderr`, `exit()`, `version_info` |
| `pathlib` | Rutas OO | `Path`, `read_text()`, `write_text()`, `glob()`, `mkdir()` |
| `shutil` | Archivos alto nivel | `copy2()`, `copytree()`, `rmtree()`, `move()`, `make_archive()` |
| `tempfile` | Archivos temporales | `NamedTemporaryFile()`, `TemporaryDirectory()`, `mkstemp()` |
| `subprocess` | Procesos externos | `run()`, `Popen()`, `check_output()`, `DEVNULL` |
| `signal` | Señales SO | `signal()`, `SIGINT`, `SIGTERM`, `SIGKILL` |
| `platform` | Info sistema | `system()`, `python_version()`, `architecture()` |

---

## 2. Textos y Datos

| Módulo | Uso Principal | Funciones Clave |
|--------|---------------|-----------------|
| `str` / `string` | Texto | `split()`, `join()`, `replace()`, `strip()`, `format()` |
| `re` | Regex | `search()`, `match()`, `findall()`, `sub()`, `finditer()`, `compile()` |
| `textwrap` | Formateo | `wrap()`, `fill()`, `dedent()`, `indent()`, `shorten()` |
| `difflib` | Diff | `unified_diff()`, `SequenceMatcher()`, `get_close_matches()` |
| `unicodedata` | Unicode | `normalize()`, `category()`, `name()`, `lookup()` |
| `html` | HTML escape | `escape()`, `unescape()` |
| `urllib.parse` | URLs | `quote()`, `unquote()`, `urlencode()`, `parse_qs()`, `urljoin()` |
| `base64` | Codificación | `b64encode()`, `b64decode()`, `urlsafe_b64encode()` |
| `hashlib` | Hash | `sha256()`, `md5()`, `blake2b()`, `pbkdf2_hmac()` |
| `hmac` | HMAC | `new()`, `compare_digest()` |
| `secrets` | Cripto seguro | `token_hex()`, `token_urlsafe()`, `choice()`, `randbits()` |
| `uuid` | UUIDs | `uuid4()`, `uuid1()`, `uuid5()`, `UUID()` |
| `binascii` | Binario-ASCII | `hexlify()`, `unhexlify()`, `a2b_base64()`, `b2a_hex()` |

---

## 3. Estructuras de Datos y Algoritmos

| Módulo | Uso Principal | Funciones Clave |
|--------|---------------|-----------------|
| `collections` | Estructuras extra | `defaultdict`, `Counter`, `deque`, `OrderedDict`, `namedtuple`, `ChainMap` |
| `itertools` | Iteradores | `chain()`, `cycle()`, `islice()`, `permutations()`, `combinations()`, `groupby()`, `accumulate()` |
| `functools` | Funciones alto nivel | `lru_cache()`, `partial()`, `reduce()`, `wraps()`, `singledispatch()`, `total_ordering()` |
| `operator` | Operadores func. | `itemgetter()`, `attrgetter()`, `methodcaller()`, `add()`, `mul()`, `eq()` |
| `heapq` | Heap (cola prioridad) | `heappush()`, `heappop()`, `heapify()`, `nlargest()`, `nsmallest()`, `merge()` |
| `bisect` | Búsqueda binaria | `bisect_left()`, `bisect_right()`, `insort_left()`, `insort_right()` |
| `array` | Arrays tipados | `array('i', [1,2,3])` (compacto, numérico) |
| `weakref` | Referencias débiles | `ref()`, `proxy()`, `WeakValueDictionary()`, `finalize()` |
| `copy` | Copias | `copy()`, `deepcopy()` |
| `pprint` | Pretty print | `pprint()`, `pformat()` |

---

## 4. Fechas, Horas y Tiempo

| Módulo | Uso Principal | Funciones Clave |
|--------|---------------|-----------------|
| `datetime` | Fecha/hora | `datetime`, `date`, `time`, `timedelta`, `timezone`, `strptime()`, `strftime()`, `isoformat()` |
| `time` | Tiempo bajo nivel | `time()`, `sleep()`, `perf_counter()`, `monotonic()`, `strftime()`, `localtime()` |
| `calendar` | Calendarios | `monthrange()`, `monthcalendar()`, `isleap()`, `weekday()` |
| `zoneinfo` (3.9+) | Zonas horarias | `ZoneInfo("America/New_York")`, `ZoneInfo.key` |
| `pytz` (externo) | Zonas legacy | `timezone()`, `localize()`, `normalize()` |

---

## 5. Matemáticas y Números

| Módulo | Uso Principal | Funciones Clave |
|--------|---------------|-----------------|
| `math` | Matemáticas | `sqrt()`, `sin()`, `cos()`, `log()`, `ceil()`, `floor()`, `gcd()`, `isclose()`, `pi`, `e` |
| `statistics` | Estadística | `mean()`, `median()`, `stdev()`, `variance()`, `quantiles()`, `mode()` |
| `random` | Aleatorio | `random()`, `randint()`, `choice()`, `shuffle()`, `sample()`, `random()` |
| `secrets` | Cripto seguro | `randbelow()`, `randbits()`, `choice()`, `token_hex()`, `token_urlsafe()` |
| `decimal` | Decimal exacto | `Decimal`, `getcontext()`, `quantize()`, `ROUND_HALF_EVEN` |
| `fractions` | Fracciones | `Fraction(3, 4)`, `limit_denominator()` |
| `statistics` | Estadística | `mean()`, `median()`, `stdev()`, `variance()`, `quantiles()` |
| `numbers` | ABC numéricos | `Number`, `Complex`, `Real`, `Rational`, `Integral` |

---

## 6. Archivos, Serialización y Datos

| Módulo | Uso Principal | Funciones Clave |
|--------|---------------|-----------------|
| `json` | JSON | `dumps()`, `loads()`, `dump()`, `load()`, `JSONEncoder` |
| `pickle` | Serialización Python | `dumps()`, `loads()`, `dump()`, `load()`, `HIGHEST_PROTOCOL` |
| `csv` | CSV | `reader()`, `writer()`, `DictReader()`, `DictWriter()` |
| `sqlite3` | SQL embebido | `connect()`, `execute()`, `executemany()`, `fetchall()`, `Row` |
| `csv` | CSV | `reader()`, `writer()`, `DictReader()`, `DictWriter()` |
| `tomllib` (3.11+) / `tomli` | TOML lectura | `load()`, `loads()` |
| `tomli_w` (externo) | TOML escritura | `dump()`, `dumps()` |
| `configparser` | INI | `ConfigParser()`, `read()`, `get()`, `sections()` |
| `xml.etree.ElementTree` | XML | `parse()`, `fromstring()`, `tostring()`, `find()`, `xpath()` |
| `xml.dom.minidom` | DOM XML | `parse()`, `prettyxml()` |
| `html.parser` | HTML parser | `HTMLParser()`, `feed()`, `handle_starttag()` |
| `email` | Email MIME | `Message`, `MIMEText`, `MIMEMultipart`, `sendmail()` |
| `mailbox` | Buzones | `mbox`, `Maildir`, `Babyl` |
| `base64` | Base64 | `b64encode()`, `b64decode()`, `urlsafe_b64encode()` |
| `binascii` | Binario-ASCII | `hexlify()`, `unhexlify()`, `crc32()`, `crc_hqx()` |
| `hashlib` | Hash | `sha256()`, `sha3_256()`, `blake2b()`, `pbkdf2_hmac()`, `shake_256()` |
| `hmac` | HMAC | `new()`, `compare_digest()` |

---

## 7. Concurrencia y Paralelismo

| Módulo | Uso Principal | Funciones Clave |
|--------|---------------|-----------------|
| `threading` | Hilos | `Thread`, `Lock`, `RLock`, `Condition`, `Semaphore`, `Event`, `Barrier`, `Timer`, `local()` |
| `multiprocessing` | Procesos | `Process`, `Pool`, `Queue`, `Pipe`, `Value`, `Array`, `Manager`, `Lock` |
| `concurrent.futures` | Alto nivel | `ThreadPoolExecutor`, `ProcessPoolExecutor`, `as_completed()`, `wait()` |
| `asyncio` | Async/await | `run()`, `create_task()`, `gather()`, `sleep()`, `Queue`, `Lock`, `StreamReader/Writer` |
| `queue` | Colas thread-safe | `Queue`, `LifoQueue`, `PriorityQueue`, `SimpleQueue` |
| `selectors` | I/O multiplexado | `DefaultSelector`, `register()`, `select()` |
| `select` | Select clásico | `select()`, `poll()`, `epoll()`, `kqueue()` |
| `signal` | Señales | `signal()`, `set_wakeup_fd()` |
| `contextvars` | Variables contexto | `ContextVar`, `copy_context()`, `run()` |

---

## 8. Redes y Protocolos

| Módulo | Uso Principal | Funciones Clave |
|--------|---------------|-----------------|
| `socket` | Sockets TCP/UDP | `socket()`, `bind()`, `listen()`, `accept()`, `connect()`, `send()`, `recv()` |
| `ssl` | TLS/SSL | `create_default_context()`, `wrap_socket()`, `create_connection()` |
| `http.client` | HTTP cliente | `HTTPConnection`, `HTTPSConnection`, `request()`, `getresponse()` |
| `urllib.request` | URL opener | `urlopen()`, `Request`, `build_opener()`, `ProxyHandler` |
| `urllib.parse` | URL parsing | `urlparse()`, `urljoin()`, `urlencode()`, `parse_qs()` |
| `urllib.robotparser` | robots.txt | `RobotFileParser()`, `can_fetch()` |
| `http.server` | HTTP servidor | `HTTPServer`, `BaseHTTPRequestHandler`, `SimpleHTTPRequestHandler` |
| `xmlrpc.client/server` | XML-RPC | `ServerProxy()`, `SimpleXMLRPCServer` |
| `ftplib` | FTP | `FTP()`, `login()`, `retrbinary()`, `storbinary()` |
| `poplib` / `imaplib` / `smtplib` | Email | `SMTP()`, `send_message()`, `IMAP4()`, `POP3()` |
| `nntplib` | NNTP | `NNTP()`, `group()`, `article()` |
| `telnetlib` | Telnet | `Telnet()`, `read_until()`, `write()` |

---

## 9. Testing, Debugging y Calidad

| Módulo | Uso Principal | Funciones Clave |
|--------|---------------|-----------------|
| `unittest` | Testing | `TestCase`, `assertEqual()`, `assertRaises()`, `mock.patch()`, `main()` |
| `doctest` | Tests en docstrings | `testmod()`, `testfile()`, `run_docstring_examples()` |
| `traceback` | Tracebacks | `print_exc()`, `format_exc()`, `extract_tb()`, `format_exception()` |
| `pdb` | Debugger | `set_trace()`, `post_mortem()`, `run()`, `runcall()` |
| `breakpoint()` (3.7+) | Breakpoint | `breakpoint()` |
| `logging` | Logging | `basicConfig()`, `getLogger()`, `debug()`, `info()`, `warning()`, `error()`, `FileHandler` |
| `warnings` | Advertencias | `warn()`, `filterwarnings()`, `catch_warnings()`, `DeprecationWarning` |
| `trace` | Trazado ejecución | `trace()`, `run()`, `runfunc()` |
| `timeit` | Benchmarking | `timeit()`, `repeat()`, `Timer()` |
| `profile` / `cProfile` | Profiling | `run()`, `runctx()`, `Stats()`, `pstats` |
| `tracemalloc` (3.4+) | Memoria | `start()`, `take_snapshot()`, `compare_to()`, `filter_traces()` |
| `faulthandler` (3.3+) | Crash dump | `enable()`, `dump_traceback()`, `register()` |

---

## 10. Introspección, Tipos y Metaprogramación

| Módulo | Uso Principal | Funciones Clave |
|--------|---------------|-----------------|
| `inspect` | Introspección | `signature()`, `getsource()`, `getfile()`, `getmembers()`, `isfunction()`, `isclass()` |
| `types` | Tipos dinámicos | `FunctionType`, `MethodType`, `ModuleType`, `SimpleNamespace`, `MappingProxyType` |
| `importlib` | Import dinámico | `import_module()`, `reload()`, `resources`, `metadata` |
| `pkgutil` | Utilidades pkg | `iter_modules()`, `get_data()`, `extend_path()` |
| `dataclasses` (3.7+) | Data classes | `dataclass`, `field()`, `asdict()`, `astuple()`, `replace()` |
| `typing` | Type hints | `TypeVar`, `Generic`, `Protocol`, `TypedDict`, `Literal`, `Annotated`, `get_type_hints()` |
| `enum` | Enums | `Enum`, `IntEnum`, `Flag`, `auto()`, `unique()` |
| `abc` | Clases abstractas | `ABC`, `abstractmethod`, `abstractproperty`, `ABCMeta` |
| `contextlib` | Context managers | `contextmanager()`, `closing()`, `suppress()`, `redirect_stdout()`, `AsyncExitStack` |
| `functools` | Decoradores | `lru_cache()`, `wraps()`, `singledispatch()`, `partial()`, `cached_property` (3.8+) |

---

## 11. Internacionalización y Localización

| Módulo | Uso Principal | Funciones Clave |
|--------|---------------|-----------------|
| `gettext` | i18n | `translation()`, `gettext()`, `ngettext()`, `install()` |
| `locale` | Locale | `setlocale()`, `getlocale()`, `localeconv()`, `strxfrm()` |
| `calendar` | Calendarios locales | `LocaleTextCalendar()`, `LocaleHTMLCalendar()` |

---

## 12. Desarrollo Web y APIs

| Módulo | Uso Principal | Funciones Clave |
|--------|---------------|-----------------|
| `wsgiref` | WSGI ref | `simple_server`, `validate`, `headers` |
| `http.cookies` | Cookies | `SimpleCookie`, `Morsel` |
| `http.cookiejar` | Cookie jar | `CookieJar`, `MozillaCookieJar`, `LWPCookieJar` |
| `urllib.robotparser` | robots.txt | `RobotFileParser()` |
| `uuid` | UUIDs | `uuid4()`, `uuid5()`, `UUID()` |

---

## 13. Multimedia y Formatos

| Módulo | Uso Principal | Funciones Clave |
|--------|---------------|-----------------|
| `wave` | WAV audio | `open()`, `getparams()`, `readframes()`, `writeframes()` |
| `aifc` | AIFF audio | `open()`, `readframes()`, `writeframes()` |
| `sunau` | Sun AU | `open()`, `readframes()` |
| `audiodev` | Dispositivos audio | (bajo nivel) |
| `colorsys` | Conversión color | `rgb_to_hsv()`, `hsv_to_rgb()`, `rgb_to_hls()` |
| `imghdr` | Detectar formato imagen | `what()`, `test()` |
| `sndhdr` | Detectar formato audio | `what()`, `test()` |

---

## 14. Compresión y Archivos

| Módulo | Uso Principal | Funciones Clave |
|--------|---------------|-----------------|
| `zlib` | Compresión zlib | `compress()`, `decompress()`, `crc32()`, `adler32()` |
| `gzip` | GZIP | `open()`, `compress()`, `decompress()`, `GzipFile` |
| `bz2` | BZIP2 | `open()`, `compress()`, `decompress()`, `BZ2File` |
| `lzma` | LZMA/XZ | `open()`, `compress()`, `decompress()`, `LZMAFile` |
| `zipfile` | ZIP | `ZipFile()`, `extractall()`, `writestr()`, `ZipInfo` |
| `tarfile` | TAR | `open()`, `extractall()`, `add()`, `TarInfo` |
| `shutil` | Archivos alto nivel | `make_archive()`, `unpack_archive()`, `get_archive_formats()` |

---

## 15. Criptografía (Built-in + `cryptography` externo)

| Módulo | Uso Principal | Funciones Clave |
|--------|---------------|-----------------|
| `hashlib` | Hash | `sha256()`, `sha3_256()`, `blake2b()`, `pbkdf2_hmac()` |
| `hmac` | HMAC | `new()`, `compare_digest()` |
| `secrets` | Cripto seguro | `token_bytes()`, `token_hex()`, `token_urlsafe()`, `randbits()` |
| `ssl` | TLS/SSL | `create_default_context()`, `wrap_socket()`, `create_connection()` |
| `cryptography` (externo) | Crypto moderno | `Fernet`, `AES`, `RSA`, `ECDSA`, `HKDF`, `X25519`, `ChaCha20Poly1305` |

---

## 16. Referencia Rápida: "¿Qué módulo uso?"

| Necesidad | Módulo |
|-----------|--------|
| Parsear fecha string | `datetime.strptime()` |
| Diferencia entre fechas | `timedelta` |
| Zona horaria | `zoneinfo.ZoneInfo` |
| Timestamp actual | `time.time()` / `datetime.now()` |
| Sleep no bloqueante (async) | `asyncio.sleep()` |
| Medir tiempo preciso | `time.perf_counter()` |
| Hash contraseña | `hashlib.pbkdf2_hmac()` / `bcrypt` (externo) |
| Token seguro | `secrets.token_urlsafe()` |
| UUID aleatorio | `uuid.uuid4()` |
| Leer CSV | `csv.DictReader()` |
| Escribir JSON | `json.dump(obj, f, indent=2)` |
| Leer archivo completo | `Path.read_text()` |
| Escribir archivo atómico | `Path.write_text()` + rename |
| Iterar directorio | `Path.rglob("*")` |
| Crear directorio recursivo | `Path.mkdir(parents=True, exist_ok=True)` |
| Hash archivo | `hashlib.file_digest(f, "sha256").hexdigest()` (3.11+) |
| Ejecutar comando | `subprocess.run(cmd, capture_output=True, text=True)` |
| Timeout comando | `subprocess.run(cmd, timeout=30)` |
| Paralelizar CPU-bound | `ProcessPoolExecutor` |
| Paralelizar I/O-bound | `ThreadPoolExecutor` / `asyncio` |
| Cache función | `@functools.lru_cache(maxsize=128)` |
| Decorador con args | `def deco(*args, **kwargs): return functools.wraps(fn)(wrapper)` |
| Singleton | `@functools.lru_cache(maxsize=None)` en factory |
| Enum con valores | `class Color(Enum): RED = 1; BLUE = 2` |
| Dataclass inmutable | `@dataclass(frozen=True)` |
| Validar tipos runtime | `typing.get_type_hints()` / `pydantic` (externo) |

---

## 17. Referencias Cruzadas

- [Fundamentos y Sintaxis](./Python%20-%20Fundamentos%20y%20Sintaxis.md) - Sintaxis base
- [Estructuras de Datos](./Python%20-%20Estructuras%20de%20Datos.md) - Tipos de collections
- [Manejo de Archivos](./Python%20-%20Manejo%20de%20Archivos%20y%20Serializaci%C3%B3n.md) - I/O, serialización
- [Concurrencia](./Python%20-%20Concurrencia%20Threading%20Multiprocessing%20Asyncio.md) - Threading, asyncio
- [Redes y HTTP](./Python%20-%20Redes%20y%20HTTP.md) - Requests, sockets
- [Testing y Calidad](./Python%20-%20Testing%2C%20Calidad%20y%20Herramientas.md) - unittest, pytest, logging

---

*Etiquetas: #python #stdlib #batteries-included #os #sys #pathlib #datetime #collections #itertools #functools #asyncio #threading #multiprocessing #json #csv #sqlite #logging #unittest #inspect #typing #dataclasses #enum #contextlib #hashlib #secrets #subprocess #subprocess #socket #ssl #urllib #email #gzip #zipfile #tarfile #sqlite3 #argparse #textwrap #difflib #unicodedata #html #urllib.parse #base64 #hashlib #hmac #uuid #binascii #statistics #fractions #decimal #numbers #calendar #zoneinfo #time #contextlib #inspect #importlib #pkgutil #dataclasses #typing #enum #abc #numbers #cmath #array #weakref #copy #pprint #doctest #traceback #pdb #breakpoint #warnings #timeit #cProfile #tracemalloc #faulthandler #gettext #locale #wsgiref #http.cookies #http.cookiejar #urllib.robotparser #wave #colorsys #imghdr #sndhdr #zlib #gzip #bz2 #lzma #zipfile #tarfile*