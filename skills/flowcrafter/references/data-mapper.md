# DataMapper (`wundii/data-mapper`)

Typisierter Object-Mapper für PHP 8.2+. Wandelt Daten aus JSON, Array, XML, CSV, YAML, NEON und Objekten in typisierte PHP-Objekte um — inklusive verschachtelter Objekte, Arrays von Objekten, DateTime und Enums.

## Wann verwenden

Bei komplexen DataMessages oder DTOs mit vielen Properties oder verschachtelten Objekten. Ersetzt manuelles Array-Zugreifen (`$data['current']['temp']`) durch typsicheres, automatisches Mapping.

**Typischer Flowcrafter-Einsatz:** HTTP-Response → DTO im Step:

```php
class FetchWeatherDataStep implements StepInterface
{
    public function __construct(
        private readonly WeatherRequestMessage $requestMessage,
        private readonly HttpClientInterface $httpClient,
        private readonly DataMapper $dataMapper,
    ) {}

    public function process(): MessageDataInterface
    {
        $response = $this->httpClient->request('GET', $url);
        $weather = $this->dataMapper->json(
            $response->getContent(),
            CurrentWeather::class,
            ['current'],  // rootElementTree — startet Mapping ab JSON-Key "current"
        );

        return new WeatherDataMessage($weather);
    }
}
```

## API-Übersicht

```php
$dataMapper = new DataMapper();

// Format-spezifische Methoden
$dto = $dataMapper->json($jsonString, MyDto::class);
$dto = $dataMapper->array($array, MyDto::class);
$dto = $dataMapper->xml($xmlString, MyDto::class);
$dto = $dataMapper->csv($csvContent, MyDto::class);
$dto = $dataMapper->yaml($yamlString, MyDto::class);

// Mit rootElementTree — startet Mapping ab einem verschachtelten Key
$dto = $dataMapper->json($json, MyDto::class, ['data']);           // json['data']
$dto = $dataMapper->json($json, MyDto::class, ['data', 'items']); // json['data']['items']

// forceInstance — leere Instanz erzeugen wenn Daten fehlen
$dto = $dataMapper->json($json, MyDto::class, [], forceInstance: true);
```

**Return-Type:** `class-string<T>` als Object-Parameter → einzelnes Objekt zurück. Instanz `T` als Object-Parameter → Array von Objekten zurück.

## DataConfig

```php
use Wundii\DataMapper\DataConfig;
use Wundii\DataMapper\Enum\ApproachEnum;
use Wundii\DataMapper\Enum\AccessibleEnum;

$dataConfig = new DataConfig(
    approachEnum: ApproachEnum::CONSTRUCTOR,
    accessibleEnum: AccessibleEnum::PUBLIC,
    classMap: [
        DateTimeInterface::class => DateTimeImmutable::class,
    ],
);
$dataMapper = new DataMapper($dataConfig);
```

### ApproachEnum

| Wert          | Verhalten                                      | Empfohlen für                     |
|---------------|------------------------------------------------|-----------------------------------|
| `CONSTRUCTOR` | Mappt auf Constructor-Parameter                | `readonly class` DTOs (Flowcrafter-Standard) |
| `PROPERTY`    | Mappt auf öffentliche Properties               | Mutable Objekte                   |
| `SETTER`      | Mappt auf Setter-Methoden (Default)            | Objekte mit Setter-Pattern        |

**Für Flowcrafter:** `ApproachEnum::CONSTRUCTOR` verwenden — passt zu `readonly class` DTOs/Messages.

### AccessibleEnum

| Wert        | Verhalten                          |
|-------------|-------------------------------------|
| `PUBLIC`    | Nur öffentliche Members (Default)  |
| `PROTECTED` | Auch geschützte Members            |
| `PRIVATE`   | Alle Members                       |

### classMap

Interface-zu-Klasse-Mapping für abstrakte Typen:

```php
classMap: [
    DateTimeInterface::class => DateTimeImmutable::class,
]
```

## TargetData-Attribut — Feld-Aliase

Wenn der Source-Key anders heißt als die Property:

```php
use Wundii\DataMapper\Attribute\TargetData;

final readonly class CurrentWeather
{
    public function __construct(
        #[TargetData('feels_like')]
        public float $feelsLike,
        #[TargetData('wind_speed')]
        public float $windSpeed,
        #[TargetData('wind_deg')]
        public int $windDeg,
    ) {}
}
```

`#[TargetData('feels_like')]` → Source-Key `feels_like` wird auf Property `$feelsLike` gemappt. Funktioniert auf Constructor-Parametern, Properties und Settern.

## Unterstützte Typen

- Skalare: `null`, `bool`, `int`, `float`, `string` (jeweils auch nullable)
- Arrays: `int[]`, `float[]`, `string[]`, `object[]`
- Objekte: verschachtelt, nullable
- Enums (backed und unit)
- DateTime/DateTimeImmutable (ISO-String oder serialisierte Darstellung)

## Namespace & Import

```php
use Wundii\DataMapper\DataMapper;
use Wundii\DataMapper\DataConfig;
use Wundii\DataMapper\Enum\ApproachEnum;
use Wundii\DataMapper\Enum\AccessibleEnum;
use Wundii\DataMapper\Attribute\TargetData;
```
