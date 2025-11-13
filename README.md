# Система расчета защитных лесных насаждений

## Описание

Программный комплекс для автоматизированного расчета размещения защитных лесных насаждений (лесополос) на территории регионов Российской Федерации. Система выполняет комплексный анализ территории с учетом природно-климатических условий, землепользования, ограничений и существующей инфраструктуры.

## Основные возможности

- Подготовка слоя ограничений - анализ и объединение различных типов территориальных ограничений
- Расчет защитных лесополос - автоматизированное определение мест размещения
- Классификация лесополос - разделение на основные, прибалочные, второстепенные и придорожные
- Учет природно-климатических условий - интеграция данных о зонах растительности, континентальности, рельефе
- Векторизация данных LULC - классификация земель по типам использования
- Геоморфометрический анализ - расчет уклонов, TPI (Topographic Position Index)

## Требования

### Зависимости

- Python 3.8+
- PostgreSQL с PostGIS
- geopandas, rasterio, shapely, sqlalchemy
- whitebox / whitebox_workflows
- networkx, centerline
- numpy, scipy, pandas, pyproj

### Установка

```bash
# Клонирование репозитория
git clone <repository-url>
cd pkp_forest_belts

# Установка uv (если еще не установлен)
# Windows (PowerShell)
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"
# Linux/Mac
curl -LsSf https://astral.sh/uv/install.sh | sh

# Синхронизация окружения (создает .venv и устанавливает все зависимости)
uv sync

# Активация окружения (опционально, uv run работает без активации)
# Linux/Mac
source .venv/bin/activate
# Windows
.venv\Scripts\activate
```

### Запуск скрипта

С использованием `uv run` (рекомендуется, не требует активации окружения):

```bash
# Просмотр справки
uv run pkp_forest_belts.py --help

# Подготовка ограничений
uv run pkp_forest_belts.py prepare_limitations --help
uv run pkp_forest_belts.py prepare_limitations --region "Липецкая область"

# Расчет лесополос
uv run pkp_forest_belts.py calculate_forest_belt --help
uv run pkp_forest_belts.py calculate_forest_belt --region "Липецкая область"
```

Или с активированным окружением:

```bash
python pkp_forest_belts.py --help
python pkp_forest_belts.py prepare_limitations --region "Липецкая область"
```

## Использование

### Интерфейс командной строки

#### 1. Подготовка слоя ограничений

```bash
uv run pkp_forest_belts.py prepare_limitations [OPTIONS]
```

Подготовка комплексного слоя ограничений для размещения защитных лесных насаждений.

**Основные параметры:**

- `--postgres-info` - путь к файлу с параметрами подключения к PostgreSQL (по умолчанию: `.secret/.gdcdb`)
- `--region` - название региона для обработки (по умолчанию: `Липецкая область`)
- `--regions-table` - полное имя таблицы с границами регионов (по умолчанию: `admin.hse_russia_regions`)
- `--region-buf-size` - размер буфера вокруг границы региона в метрах (по умолчанию: `5000`)
- `--osm-transport-table` - полное имя таблицы с транспортными объектами OSM для аэродромов (по умолчанию: `osm.gis_osm_transport_a_free`)
- `--slope-threshold` - пороговое значение уклона в градусах (по умолчанию: `12`)
- `--fabdem-zip-path` - путь к директории с ZIP-архивами тайлов FABDEM

**Примеры:**

```bash
# С параметрами по умолчанию
uv run pkp_forest_belts.py prepare_limitations

# Для другого региона
uv run pkp_forest_belts.py prepare_limitations --region "Воронежская область"

# С изменением порога уклона
uv run pkp_forest_belts.py prepare_limitations --slope-threshold 15

# Просмотр всех параметров
uv run pkp_forest_belts.py prepare_limitations --help
```

**Результат:** Файл `result/{region_shortname}_limitations.gpkg` со слоем ограничений.

---

#### 2. Расчет защитных лесных насаждений

```bash
uv run pkp_forest_belts.py calculate_forest_belt [OPTIONS]
```

Расчет защитных лесных насаждений для указанного региона.

**Основные параметры:**

- `--region` - название региона (по умолчанию: `Липецкая область`)
- `--lulc-link` - путь к растру LULC (по умолчанию: `lulc/S2LULC_10m_LAEA_48_202507081046.tif`)
- `--limitation-all-gpkg` - путь к GeoPackage с ограничениями (опционально)
- `--tpi-threshold` - пороговое значение TPI (по умолчанию: `-2`)
- `--tpi-window-size-m` - размер окна для расчета TPI в метрах (по умолчанию: `2000`)
- `--slope-threshold` - пороговое значение уклона в градусах (по умолчанию: `12`)

**Примеры:**

```bash
# С параметрами по умолчанию
uv run pkp_forest_belts.py calculate_forest_belt

# Для другого региона
uv run pkp_forest_belts.py calculate_forest_belt --region "Воронежская область"

# С готовым слоем ограничений
uv run pkp_forest_belts.py calculate_forest_belt \
    --limitation-all-gpkg "result/Lipetskaya_limitations.gpkg" \
    --limitation-all-layer "Lipetskaya_all_limitations"

# Просмотр всех параметров
uv run pkp_forest_belts.py calculate_forest_belt --help
```

**Результаты:** Файлы GeoPackage в директории `result/` с различными слоями лесополос.

---

## Описание основных функций

### prepare_limitations()

Подготовка комплексного слоя ограничений.

**Алгоритм:**
1. Подготовка водных ограничений (реки, озера, водоохранные зоны)
2. Подготовка ограничений по заболоченным территориям
3. Подготовка ограничений по неподходящим почвам
4. Подготовка ограничений по аэродромам (аэропорты и аэрополя)
5. Расчет ограничений по крутым склонам (>12°)
6. Подготовка ограничений по населенным пунктам
7. Подготовка ограничений по ООПТ
8. Подготовка ограничений по существующим лесам
9. Объединение всех ограничений в единый слой

**Возвращает:** GeoDataFrame с объединенными ограничениями

---

### calculate_forest_belt()

Расчет защитных лесных насаждений.

**Этапы:**
1. Векторизация LULC - классификация земель
2. Буферизация лесов (50 м)
3. Буферизация дорог и железных дорог
4. Объединение ограничений
5. Буферизация пашни (20 м)
6. Устранение дыр в буферных зонах
7. Построение центральных линий
8. Классификация на основные/прибалочные
9. Расчет зон облесения
10. Второстепенные лесополосы
11. Придорожные лесополосы
12. Природная классификация
13. Финальная классификация

**Возвращает:** Кортеж из двух GeoDataFrame (forest_belt_nature, forest_belt_final)

---

### Вспомогательные функции подготовки ограничений

#### prepare_water_limitations()
Подготовка ограничений по водным объектам (реки, озера, водоохранные зоны).

#### prepare_slope_limitations()
Подготовка ограничений по крутым склонам с использованием FABDEM и Whitebox Tools.

#### prepare_wetlands_limitations()
Подготовка ограничений по заболоченным территориям из OpenStreetMap.

#### prepare_soil_limitations()
Подготовка ограничений по неподходящим почвам (болотные, солончаки).

#### prepare_aerodrome_limitations()
Подготовка ограничений по аэродромам (аэропорты и аэрополя).

#### prepare_settlements_limitations()
Подготовка ограничений по населенным пунктам (НСПД + OpenStreetMap).

#### prepare_oopt_limitations()
Подготовка ограничений по особо охраняемым природным территориям.

#### prepare_forest_limitations()
Подготовка ограничений по существующим лесным массивам.

#### prepare_road_limitations()
Подготовка ограничений по дорогам с учетом их категорий (буферы 70-80 м).

#### prepare_railway_limitations()
Подготовка ограничений по железным дорогам (буфер 100 м).

---

### Функции обработки LULC и расчета лесополос

#### belt_vectorize_lulc()
Векторизация растра LULC с классификацией на воду, лес, луга, пашню, застройку.

#### belt_calculate_forest_buffer()
Расчет 50-метровых буферных зон вокруг существующих лесов.

#### belt_merge_limitation_full()
Объединение всех типов ограничений в единый слой.

#### belt_calculate_arable_buffer()
Расчет 20-метровых буферных зон вокруг пахотных земель (>10 га).

#### belt_calculate_arable_buffer_eliminate()
Устранение небольших дыр в буферных зонах пашни.

#### belt_calculate_centerlines()
Построение центральных линий полигонов для размещения лесополос.

#### belt_classify_main_gulch()
Классификация лесополос на основные (полезащитные) и прибалочные.

#### belt_calculate_forestation()
Расчет зон сплошного облесения на основе TPI, уклонов и наличия лугов.

#### belt_calculate_secondary_belt()
Расчет второстепенных лесополос вдоль дорог (буфер 6 м).

#### belt_calculate_road_belt()
Расчет придорожных лесополос вдоль основных дорог (буферы 3-7.5 м).

#### belt_forest_belt_nature()
Формирование итогового слоя с природными характеристиками и индексом Сбербанка.

#### belt_forest_belt_final()
Формирование финального слоя с зонами растительности, континентальностью и рекомендациями.

---

### Геометрические функции

#### calculate_tpi_custom_window()
Расчет TPI (Topographic Position Index) с заданным размером окна.

#### calculate_geod_buffers()
Расчет геодезических буферов с учетом кривизны Земли (UTM, AEQD).

#### gaussian_smooth()
Гауссово сглаживание геометрий (полигонов и линий).

#### chaikin_smooth()
Сглаживание геометрий алгоритмом Чайкина.

#### longest_route_from_multilines()
Поиск длиннейшего маршрута в сети линий (для деревьев и графов с циклами).

#### remove_hanging_nodes()
Удаление висячих узлов (коротких ответвлений) из сети линий.

#### connect_cluster_parts()
Соединение близких частей мультиполигона коридорами.

#### drop_small_holes()
Удаление маленьких дыр из полигонов.

#### get_region_shortname()
Получение короткого английского названия региона для именования файлов.

---

## Структура проекта

```
pkp_forest_belts/
├── pkp_forest_belts.py      # Основной модуль
├── README.md                 # Документация
├── requirements.txt          # Зависимости Python
├── .secret/                 # Конфиденциальные данные
│   └── .gdcdb              # Параметры подключения к PostgreSQL
├── lulc/                    # Растры LULC
├── result/                  # Результаты расчетов
└── .venv/                   # Виртуальное окружение
```

## Формат данных

### Конфигурация подключения к PostgreSQL

Файл `.secret/.gdcdb` содержит параметры подключения к базе данных PostgreSQL в формате JSON:

```json
{
    "host": "your-database-host.com",
    "port": 5432,
    "database": "your_database_name",
    "user": "your_username",
    "password": "your_password"
}
```

**Параметры:**
- `host` - адрес сервера PostgreSQL (IP или доменное имя)
- `port` - порт подключения (по умолчанию 5432)
- `database` - имя базы данных
- `user` - имя пользователя PostgreSQL
- `password` - пароль пользователя

**Создание файла:**

```bash
# Создайте директорию для конфиденциальных данных
mkdir .secret

# Создайте файл с параметрами подключения
# Linux/Mac
cat > .secret/.gdcdb << EOF
{
    "host": "localhost",
    "port": 5432,
    "database": "geodata",
    "user": "postgres",
    "password": "your_password"
}
EOF

# Windows (PowerShell)
@"
{
    "host": "localhost",
    "port": 5432,
    "database": "geodata",
    "user": "postgres",
    "password": "your_password"
}
"@ | Out-File -FilePath .secret\.gdcdb -Encoding utf8
```

**Важно:** 
- Файл `.secret/.gdcdb` должен быть добавлен в `.gitignore` для предотвращения утечки учетных данных
- База данных должна иметь установленное расширение PostGIS
- Пользователь должен иметь права на чтение таблиц и создание временных таблиц

---

### PostgreSQL таблицы

Все таблицы должны содержать поле `geom` с геометрией в системе координат EPSG:4326 (кроме НСПД - EPSG:3857).

#### Регионы (`admin.hse_russia_regions`)
**Обязательные поля:**
- `region` (text) - название региона (например, "Липецкая область")
- `geom` (geometry) - геометрия границы региона

#### Водные объекты линейные (`osm.gis_osm_waterways_free`)
**Обязательные поля:**
- `fclass` (text) - класс объекта (river, stream, canal и др.)
- `name` (text) - название водного объекта
- `osm_id` (bigint) - идентификатор OSM
- `geom` (geometry, LineString/MultiLineString) - геометрия водотока

#### Водные объекты полигональные (`osm.gis_osm_water_a_free`)
**Обязательные поля:**
- `fclass` (text) - класс объекта (water, reservoir, wetland и др.)
- `name` (text) - название водного объекта
- `osm_id` (bigint) - идентификатор OSM
- `geom` (geometry, Polygon/MultiPolygon) - геометрия водоема

#### Заболоченные территории (`osm.osm_wetlands_russia_final`)
**Обязательные поля:**
- `geom` (geometry, Polygon/MultiPolygon) - геометрия болота

#### Почвы (`egrpr_esoil_ru.soil_map_m2_5_v`)
**Обязательные поля:**
- `soil0` (integer) - код типа почвы (фильтруются значения 163-171 для болотных и солончаковых почв)
- `geom` (geometry, Polygon/MultiPolygon) - геометрия почвенного контура

#### Транспортные объекты (`osm.gis_osm_transport_a_free`)
**Обязательные поля:**
- `fclass` (text) - класс объекта (airport, airfield и др.)
- `name` (text) - название объекта
- `geom` (geometry, Polygon/MultiPolygon) - геометрия транспортного объекта

#### Населенные пункты НСПД (`nspd.nspd_settlements_pol`)
**Обязательные поля:**
- `geom` (geometry, Polygon/MultiPolygon, EPSG:3857) - геометрия населенного пункта

#### Населенные пункты OSM (`osm.gis_osm_places_a_free`)
**Обязательные поля:**
- `fclass` (text) - класс объекта (исключаются: county, region, island)
- `geom` (geometry, Polygon/MultiPolygon) - геометрия населенного пункта

#### ООПТ (`ecology.pkp_oopt_russia_2024`)
**Обязательные поля:**
- `name` (text) - название ООПТ (фильтруются охотничьи угодья по маске 'охотнич')
- `actuality` (text) - статус актуальности (фильтруются по маске 'действующ')
- `is_hunter_reserve` (boolean) - флаг охотничьего резервата
- `geom` (geometry, Polygon/MultiPolygon) - геометрия ООПТ

#### Леса (`forest.pkp_forest_glf`)
**Обязательные поля:**
- `geom` (geometry, Polygon/MultiPolygon) - геометрия лесного массива

#### Дороги (`osm.gis_osm_roads_free`)
**Обязательные поля:**
- `fclass` (text) - класс дороги (motorway, trunk, primary, secondary, tertiary, unclassified, track и др.)
- `ref` (text) - номер дороги (используется для фильтрации)
- `osm_id` (bigint) - идентификатор OSM
- `geom` (geometry, LineString/MultiLineString) - геометрия дороги

#### Железные дороги (`osm.gis_osm_railways_free`)
**Обязательные поля:**
- `fclass` (text) - класс ЖД (rail и др.)
- `geom` (geometry, LineString/MultiLineString) - геометрия ЖД пути

#### FABDEM тайлы (`elevation.fabdem_v1_2_tiles`)
**Обязательные поля:**
- `tile_name` (text) - название тайла (например, N052E038)
- `geom` (geometry, Polygon) - геометрия покрытия тайла

#### Зоны растительности (`g_of_russia.rf_vegetation_zone_30mln_v`)
**Обязательные поля:**
- `gid` (integer) - идентификатор записи
- `veg_id` (integer) - идентификатор зоны растительности
- `geom` (geometry, Polygon/MultiPolygon) - геометрия зоны

#### Индекс континентальности (`climate.pkp_continentalnost_index`)
**Обязательные поля:**
- `gid` (integer) - идентификатор записи
- `ic` (numeric) - значение индекса континентальности
- `geom` (geometry, Polygon/MultiPolygon) - геометрия

#### Муниципальные районы (`sber.municipal_districts_newregion`)
**Обязательные поля:**
- `region_name` (text) - название региона
- `municipal_district_name` (text) - название муниципального района
- `oktmo` (text) - код ОКТМО
- `geom` (geometry, Polygon/MultiPolygon) - геометрия района

### Растровые данные

- **LULC** - классификация земель (10 м разрешение, LAEA проекция)
- **FABDEM** - цифровая модель рельефа (30 м разрешение)
- **Луга** - маска лугов для анализа облесения

### Выходные данные

GeoPackage файлы в директории `result/` со слоями:
- Ограничения
- Классифицированные земли (LULC)
- Основные лесополосы
- Прибалочные лесополосы
- Зоны облесения
- Второстепенные лесополосы
- Придорожные лесополосы
- Итоговые слои с классификацией

## Примечания

- Все расстояния указываются в метрах
- Уклоны указываются в градусах
- TPI рассчитывается в единицах высоты (метры)
- Системы координат автоматически преобразуются при необходимости
- Для больших регионов требуется значительное время обработки
- Рекомендуется использовать SSD для временных файлов

## Авторы

НИУ ВШЭ, Центр геоданных, 2025

## Лицензия

[Указать лицензию]
