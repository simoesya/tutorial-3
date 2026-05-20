# Assignment 3 - Jetpack Compose & Annotation Processing

Course: Computação Móvel / Mobile Computing (CM)
Student(s): dam15048
Date: 2026
Repository URL: ___________

---

## 1. Introduction

Este trabalho aprofunda o desenvolvimento Android e Kotlin com dois projetos distintos. O primeiro, **CoolJetpackWeatherApp**, é uma aplicação Android de meteorologia desenvolvida com **Jetpack Compose** e o cliente HTTP **Ktor**, utilizando padrão MVVM com `StateFlow`. O segundo, **GreetingProcessorProject**, é um projeto Kotlin multi-módulo que explora **annotation processing** com KAPT, gerando código em tempo de compilação através da biblioteca **KotlinPoet**.

Os objetivos principais foram: migrar de XML Views para Jetpack Compose, compreender o ciclo de vida de composables e o estado reativo com `StateFlow`, e explorar a geração de código em compile-time com processadores de anotações personalizados.

## 2. System Overview

### CoolJetpackWeatherApp
Aplicação Android nativa em Kotlin com Jetpack Compose que consulta a API Open-Meteo para obter dados meteorológicos em tempo real. O utilizador introduz as coordenadas (latitude/longitude) e pressiona "Update" para obter a temperatura, velocidade e direção do vento, código meteorológico, pressão ao nível do mar e hora da medição. A UI é totalmente declarativa com Compose e o estado é gerido por `StateFlow` no ViewModel.

### GreetingProcessorProject
Projeto Kotlin multi-módulo (Gradle) com três módulos — `annotations`, `processor` e `app` — que demonstra a geração de código em tempo de compilação. O módulo `annotations` define duas anotações (`@Greeting` e `@Extract`). O módulo `processor` implementa dois processadores KAPT que, usando KotlinPoet, geram automaticamente classes wrapper e extratoras. O módulo `app` consume as anotações e o código gerado.

## 3. Architecture and Design

### CoolJetpackWeatherApp — MVVM com StateFlow e Compose

```
app/src/main/
├── AndroidManifest.xml
└── java/com/dam15048/cooljetpackweatherapp/
    ├── MainActivity.kt                  (entry point, aplica o tema e chama WeatherUI)
    ├── data/
    │   ├── weatherData.kt               (data classes: WeatherData, CurrentWeather, Hourly)
    │   └── weatherApiClient.kt          (objeto Ktor: WeatherApiClient.getWeather())
    ├── viewmodel/
    │   ├── weatherUIState.kt            (data class WeatherUIState)
    │   └── weatherViewModel.kt          (ViewModel com MutableStateFlow)
    └── ui/theme/
        ├── weatherScreen.kt             (composables: WeatherUI, CoordinatesCard, WeatherCard, WeatherRow)
        ├── Theme.kt                     (Material3 com dynamic color)
        ├── Color.kt
        └── Type.kt

res/
├── drawable/   (ic_sun.xml, ic_cloud.xml, ic_rain.xml)
└── values/     (strings.xml, colors.xml, themes.xml)
```

Fluxo de dados:
```
WeatherApiClient (Ktor) → WeatherData → WeatherViewModel (StateFlow) → WeatherUI (Compose)
```

### GreetingProcessorProject — Multi-módulo com KAPT

```
GreetingProcessorProject/
├── settings.gradle.kts          (inclui: annotations, processor, app)
├── annotations/
│   └── src/main/kotlin/annotations/
│       ├── Greeting.kt          (@Greeting annotation)
│       └── Extract.kt           (@Extract annotation)
├── processor/
│   └── src/main/kotlin/processor/
│       ├── GreetingProcessor.kt (gera *Wrapper classes)
│       └── RegexProcessor.kt    (gera *Extractor classes)
└── app/
    └── src/main/kotlin/com/example/app/
        ├── MyClass.kt           (usa @Greeting)
        ├── DataProcessor.kt     (usa @Extract, classe abstrata)
        └── Main.kt              (usa código gerado: MyClassWrapper, DataProcessorExtractor)
```

Dependências entre módulos:
- `app` → depende de `annotations` (implementação) e `processor` (via kapt)
- `processor` → depende de `annotations` + KotlinPoet + AutoService

## 4. Implementation

### CoolJetpackWeatherApp

**weatherData.kt** — Data classes anotadas com `@Serializable` (kotlinx.serialization) que mapeiam a resposta JSON da API Open-Meteo:

```kotlin
@Serializable
data class WeatherData(
    val latitude: Float = 0f,
    val longitude: Float = 0f,
    val timezone: String = "",
    val current_weather: CurrentWeather = CurrentWeather(),
    val hourly: Hourly = Hourly()
)

@Serializable
data class CurrentWeather(
    val temperature: Float = 0f,
    val windspeed: Float = 0f,
    val winddirection: Int = 0,
    val weathercode: Int = 0,
    val time: String = ""
)
```

**weatherApiClient.kt** — Objeto singleton que configura o cliente Ktor com `ContentNegotiation` e Json lenient. A função `getWeather()` é `suspend` e devolve `WeatherData?` em caso de erro:

```kotlin
object WeatherApiClient {
    private val client = HttpClient(Android) {
        install(ContentNegotiation) {
            json(Json { isLenient = true; ignoreUnknownKeys = true })
        }
    }
    suspend fun getWeather(lat: Float, lon: Float): WeatherData? {
        return try { client.get(reqString).body() } catch (e: Exception) { null }
    }
}
```

**weatherUIState.kt** — Data class imutável que representa o estado completo da UI. O valor por omissão da latitude e longitude corresponde a Lisboa:

```kotlin
data class WeatherUIState(
    val latitude: Float = 38.736946f,
    val longitude: Float = -9.142685f,
    val temperature: Float = 0f,
    val windspeed: Float = 0f,
    val winddirection: Int = 0,
    val weathercode: Int = 0,
    val seaLevelPressure: Float = 0f,
    val time: String = ""
)
```

**weatherViewModel.kt** — Usa `MutableStateFlow<WeatherUIState>` para expor o estado de forma reativa. A função `fetchWeather()` corre dentro de `viewModelScope.launch` (coroutines), garantindo que a chamada de rede não bloqueia a main thread:

```kotlin
class WeatherViewModel : ViewModel() {
    private val _uiState = MutableStateFlow(WeatherUIState())
    val uiState: StateFlow<WeatherUIState> = _uiState.asStateFlow()

    fun fetchWeather() {
        viewModelScope.launch {
            val weather = WeatherApiClient.getWeather(_uiState.value.latitude, _uiState.value.longitude)
            if (weather != null) {
                _uiState.value = _uiState.value.copy(
                    temperature = weather.current_weather.temperature,
                    windspeed = weather.current_weather.windspeed,
                    // ...
                )
            }
        }
    }
}
```

**weatherScreen.kt** — UI totalmente declarativa com Jetpack Compose. O composable `WeatherUI` observa o `StateFlow` com `collectAsState()`. O ícone meteorológico é selecionado dinamicamente com base no `weathercode`:

```kotlin
@Composable
fun WeatherUI(weatherViewModel: WeatherViewModel = viewModel()) {
    val weatherUIState by weatherViewModel.uiState.collectAsState()
    // ...
    val icon = when (weatherUIState.weathercode) {
        0, 1 -> R.drawable.ic_sun
        2, 3 -> R.drawable.ic_cloud
        else -> R.drawable.ic_rain
    }
    Image(painter = painterResource(id = icon), ...)
    CoordinatesCard(...)
    WeatherCard(...)
}
```

A UI é composta por dois `Card`s: `CoordinatesCard` (campos de entrada + botão Update) e `WeatherCard` (grelha de dados com `WeatherRow`s).

**MainActivity.kt** — Minimalista: habilita edge-to-edge, aplica o tema `COolJetpackWeatherAppTheme` e chama `WeatherUI()` dentro de `setContent`:

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            COolJetpackWeatherAppTheme { WeatherUI() }
        }
    }
}
```

Permissões: `INTERNET`. Dependências principais: Compose BOM, Material3, Ktor (core, android, content-negotiation, serialization-json), kotlinx-serialization-json, lifecycle-viewmodel-compose.

---

### GreetingProcessorProject

**@Greeting e @Extract (annotations/)** — Anotações de source retention que anotam funções:

```kotlin
@Target(AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.SOURCE)
annotation class Greeting(val message: String)

@Target(AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.SOURCE)
annotation class Extract(val regex: String)
```

**GreetingProcessor (processor/)** — Processador KAPT registado via `@AutoService`. Para cada classe com métodos anotados com `@Greeting`, gera uma classe `*Wrapper` que, antes de delegar para o método original, imprime a mensagem da anotação:

```kotlin
// Código gerado para MyClass:
class MyClassWrapper(val original: MyClass) {
    fun sayHello() {
        println("Hello from MyClass!")
        original.sayHello()
    }
    fun compute() {
        println("Welcome to the compute function!")
        original.compute()
    }
}
```

**RegexProcessor (processor/)** — Gera uma classe `*Extractor` que herda da classe abstrata anotada e implementa os métodos com `@Extract`, usando a regex da anotação para extrair grupos do input:

```kotlin
// Código gerado para DataProcessor:
class DataProcessorExtractor(input: String) : DataProcessor(input) {
    override fun getName(): String? =
        Regex("Name: (\\w+)").find(input)?.groupValues?.get(1)
    override fun getAddress(): String? =
        Regex("Address: (.+)").find(input)?.groupValues?.get(1)
}
```

**MyClass e DataProcessor (app/)** — Classes que usam as anotações:

```kotlin
open class MyClass {
    @Greeting("Hello from MyClass!")
    open fun sayHello() { println("Executing sayHello method") }

    @Greeting("Welcome to the compute function!")
    open fun compute() { println("Computing something important...") }
}

abstract class DataProcessor(val input: String) {
    @Extract(regex = "Name: (\\w+)")
    abstract fun getName(): String?

    @Extract(regex = "Address: (.+)")
    abstract fun getAddress(): String?
}
```

**Main.kt (app/)** — Demonstra o uso do código gerado em compile-time:

```kotlin
fun main() {
    val wrappedMyClass = MyClassWrapper(MyClass())
    wrappedMyClass.sayHello()
    wrappedMyClass.compute()

    val extractor = DataProcessorExtractor("Name: John Address: 123 Street")
    println("Name: ${extractor.getName()}")
    println("Address: ${extractor.getAddress()}")
}
```

Saída esperada:
```
=== Greeting Processor ===
Hello from MyClass!
Executing sayHello method
Welcome to the compute function!
Computing something important...

=== Regex Processor ===
Name: John
Address: 123 Street
```

## 5. Testing and Validation

**CoolJetpackWeatherApp:** Testado em AVD (Pixel 9 Pro, API 34). Verificado o carregamento de dados ao pressionar "Update" com as coordenadas de Lisboa por omissão, a seleção correta do ícone (sol, nuvem, chuva) consoante o `weathercode`, e a atualização reativa da UI via `StateFlow`. Incluídos `ExampleUnitTest` e `ExampleInstrumentedTest` gerados pelo Android Studio.

**GreetingProcessorProject:** Validado através da execução do `Main.kt` do módulo `app`. Verificada a geração das classes `MyClassWrapper` e `DataProcessorExtractor` durante a compilação (KAPT), a impressão das mensagens da anotação `@Greeting` antes da delegação, e a extração correta de dados com `@Extract` usando regex.

Limitações conhecidas: o CoolJetpackWeatherApp não trata o caso em que o utilizador introduz coordenadas inválidas (a conversão `toFloatOrNull()` ignora silenciosamente o input). Não há indicador de carregamento durante o fetch da API.

## 6. Usage Instructions

### CoolJetpackWeatherApp
1. Abrir o projeto em Android Studio.
2. Sincronizar o Gradle (requer compileSdk 36 e Kotlin 2.2.10+).
3. Executar num AVD (API 24+) ou dispositivo físico com Android 7.0+.
4. As coordenadas de Lisboa são preenchidas por omissão. Alterar se necessário e pressionar "Update".

### GreetingProcessorProject
1. Abrir o projeto em IntelliJ IDEA.
2. Sincronizar o Gradle (requer JVM 23+).
3. Executar `Build > Build Project` para ativar o KAPT e gerar as classes `MyClassWrapper` e `DataProcessorExtractor`.
4. Executar `app/src/main/kotlin/com/example/app/Main.kt` para ver o output.

---

# Autonomous Software Engineering Sections - only for [AC OK, AI OK] sections

## 7. Prompting Strategy

<!-- Descrever os prompts utilizados com ferramentas de IA, o seu propósito e como evoluíram. Incluir exemplos representativos. -->

## 8. Autonomous Agent Workflow

<!-- Explicar como as ferramentas de IA ou agentes contribuíram para o desenvolvimento: planeamento, codificação, debugging, testes, documentação, etc. -->

## 9. Verification of AI-Generated Artifacts

<!-- Descrever como foi verificada a correção do código/designs gerados por IA (testes, revisão manual, análise estática, etc.). -->

## 10. Human vs AI Contribution

| Componente | Contribuição |
|---|---|
| CoolJetpackWeatherApp — Toda a implementação | Humano |
| GreetingProcessorProject — Toda a implementação | Humano |
| Relatório | Humano |

## 11. Ethical and Responsible Use

<!-- Reflexão sobre riscos, limitações, enviesamentos ou outputs inapropriados das ferramentas de IA e como foram geridos. -->

---

# Development Process

## 12. Version Control and Commit History

O desenvolvimento foi realizado sob controlo de versão Git. O histórico de commits reflete a progressão incremental do trabalho, desde a configuração inicial dos projetos até à implementação final de cada componente.

## 13. Difficulties and Lessons Learned

- **Jetpack Compose vs XML Views**: a mudança de paradigma para UI declarativa exige uma nova forma de pensar o estado e a recomposição. Perceber quando um composable re-executa foi o maior desafio inicial.
- **StateFlow vs LiveData**: `StateFlow` é mais adequado para Compose por ser reativo por natureza e não requerer um `LifecycleOwner`. A função `collectAsState()` integra-se de forma limpa com o ciclo de vida dos composables.
- **Ktor vs Retrofit**: Ktor é mais idiomático em Kotlin (suspend functions nativas, sem callbacks), mas a configuração inicial é mais verbosa do que Retrofit.
- **KAPT e geração de código**: perceber o ciclo de processamento de anotações (rounds), a diferença entre `SOURCE` e `RUNTIME` retention, e como o KotlinPoet constrói a AST do ficheiro gerado foram os pontos mais complexos do GreetingProcessorProject.
- **Estrutura multi-módulo**: a separação entre `annotations`, `processor` e `app` é essencial para evitar dependências circulares e garantir que o processador não é incluído no artefacto final.

## 14. Future Improvements

- **CoolJetpackWeatherApp**: adicionar validação de input para coordenadas inválidas; mostrar `CircularProgressIndicator` durante o fetch; adicionar suporte a GPS (sem necessidade de introduzir coordenadas manualmente); exibir previsão horária num gráfico com a biblioteca Charts para Compose.
- **GreetingProcessorProject**: migrar de KAPT para **KSP** (Kotlin Symbol Processing), que é mais rápido e é o sucessor recomendado; adicionar suporte a múltiplos grupos de captura em `@Extract`; escrever testes unitários para os processadores.

---

## 15. AI Usage Disclosure (Mandatory)

Nenhuma ferramenta de IA foi utilizada no desenvolvimento dos projetos deste tutorial (as secções relevantes eram `[AC YES, AI NO]` ou `[AC NO, AI NO]`).

O presente relatório foi elaborado pelo aluno, que assume total responsabilidade pelo seu conteúdo.
