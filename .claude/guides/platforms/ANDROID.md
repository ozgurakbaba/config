# Android Development Guide

## Tech Stack

- **Languages:** Kotlin 1.9+, Java (legacy support)
- **UI Frameworks:** Jetpack Compose (primary), XML Views (legacy)
- **AR/Spatial:** ARCore, Scene Viewer, Android XR, Jetpack XR
- **IDE:** Android Studio Hedgehog+
- **Build System:** Gradle with Kotlin DSL
- **Dependency Management:** Gradle dependencies
- **Testing:** JUnit, Espresso, Compose UI Test

---

## Project Structure

```
app/
├── src/
│   ├── main/
│   │   ├── java/com/example/myapp/
│   │   │   ├── MainActivity.kt
│   │   │   ├── ui/
│   │   │   │   ├── theme/
│   │   │   │   ├── screens/
│   │   │   │   │   ├── home/
│   │   │   │   │   │   ├── HomeScreen.kt
│   │   │   │   │   │   └── HomeViewModel.kt
│   │   │   │   │   └── profile/
│   │   │   ├── data/
│   │   │   │   ├── repository/
│   │   │   │   ├── network/
│   │   │   │   └── local/
│   │   │   ├── domain/
│   │   │   │   ├── model/
│   │   │   │   └── usecase/
│   │   │   └── di/             # Dependency Injection
│   │   ├── res/
│   │   │   ├── values/
│   │   │   ├── drawable/
│   │   │   └── layout/
│   │   └── AndroidManifest.xml
│   └── test/
└── build.gradle.kts
```

---

## Jetpack Compose Best Practices

### Composable Architecture

```kotlin
// MVVM pattern with Compose
@Composable
fun ProfileScreen(
    viewModel: ProfileViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsState()

    LaunchedEffect(Unit) {
        viewModel.loadProfile()
    }

    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp)
    ) {
        Text(text = uiState.userName)
        Button(onClick = { viewModel.updateProfile() }) {
            Text("Update")
        }
    }
}

// ViewModel
class ProfileViewModel @Inject constructor(
    private val repository: ProfileRepository
) : ViewModel() {
    private val _uiState = MutableStateFlow(ProfileUiState())
    val uiState: StateFlow<ProfileUiState> = _uiState.asStateFlow()

    fun loadProfile() {
        viewModelScope.launch {
            repository.getProfile()
                .collect { profile ->
                    _uiState.update { it.copy(userName = profile.name) }
                }
        }
    }
}

data class ProfileUiState(
    val userName: String = "",
    val isLoading: Boolean = false,
    val error: String? = null
)
```

### State Management

```kotlin
// remember for simple state
var expanded by remember { mutableStateOf(false) }

// rememberSaveable for config changes
var text by rememberSaveable { mutableStateOf("") }

// derivedStateOf for computed state
val hasErrors by remember {
    derivedStateOf { text.length < 3 }
}

// StateFlow from ViewModel
val uiState by viewModel.uiState.collectAsStateWithLifecycle()
```

### Performance

```kotlin
// Use keys in lists
LazyColumn {
    items(
        items = listItems,
        key = { it.id }
    ) { item ->
        ItemRow(item = item)
    }
}

// Stable parameters to avoid recomposition
@Composable
fun ItemRow(item: Item) { // Data class is stable
    // ...
}

// Use remember for expensive calculations
val sortedList = remember(items) {
    items.sortedBy { it.name }
}
```

---

## XML Views (Legacy)

### When to Use XML Views

- Existing codebase with XML layouts
- Complex RecyclerView with multiple view types
- MotionLayout animations
- Gradual migration to Compose

### View Binding (Preferred over findViewById)

```kotlin
// build.gradle.kts
android {
    viewBinding {
        enable = true
    }
}

// Activity
class MainActivity : AppCompatActivity() {
    private lateinit var binding: ActivityMainBinding

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        binding.button.setOnClickListener {
            // Handle click
        }
    }
}
```

---

## ARCore / Android XR

### ARCore Setup

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.google.ar:core:1.41.0")
}

// AndroidManifest.xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-feature android:name="android.hardware.camera.ar" android:required="true" />

<application>
    <meta-data
        android:name="com.google.ar.core"
        android:value="required" />
</application>
```

### Basic AR Scene

```kotlin
import com.google.ar.core.*
import com.google.ar.sceneform.ArSceneView

class ArActivity : AppCompatActivity() {
    private lateinit var arSceneView: ArSceneView
    private var session: Session? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // Check AR support
        if (!ArCoreApk.getInstance().checkAvailability(this).isSupported) {
            Toast.makeText(this, "ARCore not supported", Toast.LENGTH_SHORT).show()
            finish()
            return
        }

        // Initialize AR session
        session = Session(this).apply {
            val config = Config(this).apply {
                planeFindingMode = Config.PlaneFindingMode.HORIZONTAL
                lightEstimationMode = Config.LightEstimationMode.ENVIRONMENTAL_HDR
            }
            configure(config)
        }

        arSceneView = ArSceneView(this).apply {
            setupSession(session)
        }

        setContentView(arSceneView)
    }

    override fun onResume() {
        super.onResume()
        session?.resume()
    }

    override fun onPause() {
        super.onPause()
        session?.pause()
    }
}
```

### Jetpack XR (New - Compose for XR)

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.xr:xr-compose:1.0.0-alpha01")
}

@Composable
fun XrScene() {
    XrSpatialScene {
        SpatialPanel(
            modifier = Modifier.size(width = 1.dp, height = 0.5.dp)
        ) {
            Text("Hello XR!")
        }

        // 3D Content
        GltfModel(
            source = GltfModelSource.FromAssets("models/robot.glb"),
            modifier = Modifier
                .position(x = 0.dp, y = 0.dp, z = -2.dp)
                .scale(0.5f)
        )
    }
}
```

---

## Networking

### Retrofit + Kotlin Coroutines

```kotlin
// API Interface
interface ApiService {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: String): User

    @POST("users")
    suspend fun createUser(@Body user: User): User
}

// Retrofit Setup
object RetrofitClient {
    private const val BASE_URL = "https://api.example.com/"

    val apiService: ApiService by lazy {
        Retrofit.Builder()
            .baseUrl(BASE_URL)
            .addConverterFactory(GsonConverterFactory.create())
            .client(OkHttpClient.Builder()
                .addInterceptor { chain ->
                    val request = chain.request().newBuilder()
                        .addHeader("Authorization", "Bearer $token")
                        .build()
                    chain.proceed(request)
                }
                .build())
            .build()
            .create(ApiService::class.java)
    }
}

// Repository
class UserRepository @Inject constructor(
    private val apiService: ApiService
) {
    suspend fun getUser(id: String): Result<User> = try {
        Result.success(apiService.getUser(id))
    } catch (e: Exception) {
        Result.failure(e)
    }
}

// Usage in ViewModel
viewModelScope.launch {
    repository.getUser("123")
        .onSuccess { user ->
            _uiState.update { it.copy(user = user) }
        }
        .onFailure { error ->
            _uiState.update { it.copy(error = error.message) }
        }
}
```

---

## Dependency Management

### Gradle with Kotlin DSL

```kotlin
// build.gradle.kts (Project level)
plugins {
    id("com.android.application") version "8.2.0" apply false
    id("org.jetbrains.kotlin.android") version "1.9.20" apply false
    id("com.google.dagger.hilt.android") version "2.48" apply false
}

// build.gradle.kts (App level)
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("kotlin-kapt")
    id("com.google.dagger.hilt.android")
}

dependencies {
    // Jetpack Compose
    val composeBom = platform("androidx.compose:compose-bom:2023.10.01")
    implementation(composeBom)
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.material3:material3")
    implementation("androidx.compose.ui:ui-tooling-preview")

    // Lifecycle
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.6.2")
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.6.2")

    // Hilt
    implementation("com.google.dagger:hilt-android:2.48")
    kapt("com.google.dagger:hilt-compiler:2.48")

    // Networking
    implementation("com.squareup.retrofit2:retrofit:2.9.0")
    implementation("com.squareup.retrofit2:converter-gson:2.9.0")
}
```

---

## Dependency Injection (Hilt)

```kotlin
// Application class
@HiltAndroidApp
class MyApplication : Application()

// Module
@Module
@InstallIn(SingletonComponent::class)
object AppModule {
    @Provides
    @Singleton
    fun provideApiService(): ApiService {
        return RetrofitClient.apiService
    }

    @Provides
    @Singleton
    fun provideUserRepository(apiService: ApiService): UserRepository {
        return UserRepository(apiService)
    }
}

// Activity
@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MyAppTheme {
                NavGraph()
            }
        }
    }
}

// ViewModel
@HiltViewModel
class ProfileViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel()
```

---

## Testing

### Unit Tests

```kotlin
class ViewModelTest {
    @get:Rule
    val instantExecutorRule = InstantTaskExecutorRule()

    private lateinit var viewModel: ProfileViewModel
    private lateinit var repository: FakeUserRepository

    @Before
    fun setup() {
        repository = FakeUserRepository()
        viewModel = ProfileViewModel(repository)
    }

    @Test
    fun `loadProfile updates uiState with user data`() = runTest {
        viewModel.loadProfile()

        val state = viewModel.uiState.value
        assertEquals("John Doe", state.userName)
        assertFalse(state.isLoading)
    }
}
```

### Compose UI Tests

```kotlin
class HomeScreenTest {
    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun clickButton_navigatesToProfile() {
        composeTestRule.setContent {
            HomeScreen()
        }

        composeTestRule
            .onNodeWithText("Profile")
            .performClick()

        composeTestRule
            .onNodeWithText("Profile Screen")
            .assertIsDisplayed()
    }
}
```

---

## Security

### Encrypted SharedPreferences

```kotlin
val masterKey = MasterKey.Builder(context)
    .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
    .build()

val sharedPreferences = EncryptedSharedPreferences.create(
    context,
    "secret_shared_prefs",
    masterKey,
    EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
    EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
)

sharedPreferences.edit {
    putString("api_token", token)
}
```

### Network Security Config

```xml
<!-- res/xml/network_security_config.xml -->
<network-security-config>
    <domain-config cleartextTrafficPermitted="false">
        <domain includeSubdomains="true">api.example.com</domain>
    </domain-config>
</network-security-config>

<!-- AndroidManifest.xml -->
<application
    android:networkSecurityConfig="@xml/network_security_config">
</application>
```

---

## Performance

### Baseline Profiles

```kotlin
// benchmark/src/main/java/BaselineProfileGenerator.kt
@ExperimentalBaselineProfilesApi
@RunWith(AndroidJUnit4::class)
class BaselineProfileGenerator {
    @get:Rule
    val baselineProfileRule = BaselineProfileRule()

    @Test
    fun generateBaselineProfile() = baselineProfileRule.collect(
        packageName = "com.example.myapp"
    ) {
        startActivityAndWait()
        // Navigate through key user flows
    }
}
```

### Memory Profiling

```kotlin
// Detect leaks with LeakCanary
dependencies {
    debugImplementation("com.squareup.leakcanary:leakcanary-android:2.12")
}
```

---

## Distribution

### Build Variants

```kotlin
android {
    buildTypes {
        debug {
            applicationIdSuffix = ".debug"
            isDebuggable = true
        }
        release {
            isMinifyEnabled = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }

    flavorDimensions += "environment"
    productFlavors {
        create("dev") {
            dimension = "environment"
            buildConfigField("String", "API_URL", "\"https://dev-api.example.com\"")
        }
        create("prod") {
            dimension = "environment"
            buildConfigField("String", "API_URL", "\"https://api.example.com\"")
        }
    }
}
```

### Signing Config

```kotlin
android {
    signingConfigs {
        create("release") {
            storeFile = file("../keystore.jks")
            storePassword = System.getenv("KEYSTORE_PASSWORD")
            keyAlias = System.getenv("KEY_ALIAS")
            keyPassword = System.getenv("KEY_PASSWORD")
        }
    }

    buildTypes {
        release {
            signingConfig = signingConfigs.getByName("release")
        }
    }
}
```

---

## External Resources

- **Android Developers:** https://developer.android.com/
- **Jetpack Compose:** https://developer.android.com/jetpack/compose
- **ARCore:** https://developers.google.com/ar
- **Android XR:** https://developer.android.com/xr
- **Kotlin Docs:** https://kotlinlang.org/docs/

---

## Common Gotchas

1. **Context Leaks:** Don't hold Activity context in ViewModels (use ApplicationContext)
2. **Lifecycle:** Collect StateFlow with `collectAsStateWithLifecycle()` in Compose
3. **Recomposition:** Avoid unstable parameters in Composables
4. **BackHandler:** Use `BackHandler` in Compose, not `onBackPressed()`
5. **Permissions:** Request runtime permissions for camera, location, etc.
