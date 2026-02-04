# iOS Development Guide

## Tech Stack

- **Languages:** Swift 5.9+, Objective-C (legacy support)
- **UI Frameworks:** SwiftUI (primary), UIKit (when needed)
- **AR/Spatial:** ARKit, RealityKit, visionOS
- **IDE:** Xcode 15+
- **Dependency Management:** Swift Package Manager (SPM), CocoaPods (legacy)
- **Testing:** XCTest, XCUITest

---

## Project Structure

```
MyApp/
├── MyApp/
│   ├── App/
│   │   ├── MyAppApp.swift          # App entry point
│   │   └── ContentView.swift       # Root view
│   ├── Features/
│   │   ├── Home/
│   │   │   ├── HomeView.swift
│   │   │   └── HomeViewModel.swift
│   │   └── Profile/
│   ├── Core/
│   │   ├── Network/
│   │   ├── Persistence/
│   │   └── Utilities/
│   ├── Resources/
│   │   ├── Assets.xcassets
│   │   └── Localizable.strings
│   └── Info.plist
└── MyAppTests/
```

---

## SwiftUI Best Practices

### View Architecture

```swift
// MVVM pattern - separate concerns
struct ProfileView: View {
    @StateObject private var viewModel = ProfileViewModel()

    var body: some View {
        VStack {
            Text(viewModel.userName)
            Button("Update") {
                viewModel.updateProfile()
            }
        }
        .task {
            await viewModel.loadProfile()
        }
    }
}

class ProfileViewModel: ObservableObject {
    @Published var userName: String = ""

    func loadProfile() async {
        // Network call
    }
}
```

### State Management

```swift
// @State for local view state
@State private var isExpanded = false

// @StateObject for view-owned objects
@StateObject private var viewModel = ViewModel()

// @ObservedObject for passed-in objects
@ObservedObject var sharedData: SharedData

// @EnvironmentObject for dependency injection
@EnvironmentObject var authManager: AuthManager

// @Binding for two-way binding
@Binding var selectedTab: Int
```

### Performance

```swift
// Use LazyVStack/LazyHStack for large lists
LazyVStack {
    ForEach(items) { item in
        ItemRow(item: item)
    }
}

// Avoid heavy work in body
var body: some View {
    VStack {
        expensiveView // Computed once
    }
}

private var expensiveView: some View {
    // Complex view logic
}
```

---

## UIKit Integration

### When to Use UIKit

- Complex table/collection view layouts
- Advanced text editing
- Camera/photo picker customization
- Legacy code integration
- Performance-critical scrolling

### SwiftUI + UIKit Bridge

```swift
// UIViewRepresentable for UIKit views
struct MapViewWrapper: UIViewRepresentable {
    func makeUIView(context: Context) -> MKMapView {
        let mapView = MKMapView()
        return mapView
    }

    func updateUIView(_ uiView: MKMapView, context: Context) {
        // Update view when SwiftUI state changes
    }
}

// UIViewControllerRepresentable for view controllers
struct ImagePickerWrapper: UIViewControllerRepresentable {
    @Binding var image: UIImage?

    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.delegate = context.coordinator
        return picker
    }

    func updateUIViewController(_ uiViewController: UIImagePickerController, context: Context) {}

    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }

    class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
        let parent: ImagePickerWrapper

        init(_ parent: ImagePickerWrapper) {
            self.parent = parent
        }

        func imagePickerController(_ picker: UIImagePickerController, didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey : Any]) {
            if let image = info[.originalImage] as? UIImage {
                parent.image = image
            }
        }
    }
}
```

---

## ARKit / Spatial Development

### ARKit Quick Start

```swift
import ARKit
import RealityKit

struct ARViewContainer: UIViewRepresentable {
    func makeUIView(context: Context) -> ARView {
        let arView = ARView(frame: .zero)

        // Configure AR session
        let config = ARWorldTrackingConfiguration()
        config.planeDetection = [.horizontal, .vertical]
        config.environmentTexturing = .automatic

        arView.session.run(config)

        // Add content
        let anchor = AnchorEntity(plane: .horizontal)
        let box = ModelEntity(mesh: .generateBox(size: 0.1))
        anchor.addChild(box)
        arView.scene.addAnchor(anchor)

        return arView
    }

    func updateUIView(_ uiView: ARView, context: Context) {}
}
```

### Hand Tracking (visionOS)

```swift
import ARKit

class HandTrackingManager: ObservableObject {
    @Published var leftHandPosition: SIMD3<Float>?
    @Published var rightHandPosition: SIMD3<Float>?

    func startTracking() {
        let session = ARKitSession()
        let handTracking = HandTrackingProvider()

        Task {
            try await session.run([handTracking])

            for await update in handTracking.anchorUpdates {
                switch update.anchor.chirality {
                case .left:
                    leftHandPosition = update.anchor.handSkeleton?.joint(.wrist).anchorFromJointTransform.columns.3.xyz
                case .right:
                    rightHandPosition = update.anchor.handSkeleton?.joint(.wrist).anchorFromJointTransform.columns.3.xyz
                }
            }
        }
    }
}
```

---

## Networking

### Modern Async/Await Pattern

```swift
// API Client
actor APIClient {
    private let session: URLSession
    private let baseURL: URL

    init(baseURL: URL) {
        self.baseURL = baseURL
        self.session = URLSession.shared
    }

    func request<T: Decodable>(_ endpoint: String) async throws -> T {
        let url = baseURL.appendingPathComponent(endpoint)
        let (data, response) = try await session.data(from: url)

        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            throw APIError.invalidResponse
        }

        return try JSONDecoder().decode(T.self, from: data)
    }
}

enum APIError: Error {
    case invalidResponse
    case decodingError
}

// Usage
Task {
    let client = APIClient(baseURL: URL(string: "https://api.example.com")!)
    let user: User = try await client.request("users/me")
}
```

---

## Dependency Management

### Swift Package Manager (Preferred)

```swift
// Package.swift
dependencies: [
    .package(url: "https://github.com/Alamofire/Alamofire.git", from: "5.8.0"),
    .package(url: "https://github.com/onevcat/Kingfisher.git", from: "7.10.0")
]

// In Xcode: File > Add Package Dependencies
```

### CocoaPods (Legacy)

```ruby
# Podfile
platform :ios, '15.0'
use_frameworks!

target 'MyApp' do
  pod 'Alamofire', '~> 5.8'
  pod 'Kingfisher', '~> 7.10'
end
```

---

## Testing

### Unit Tests

```swift
import XCTest
@testable import MyApp

final class ViewModelTests: XCTestCase {
    var viewModel: ProfileViewModel!

    override func setUp() {
        super.setUp()
        viewModel = ProfileViewModel()
    }

    override func tearDown() {
        viewModel = nil
        super.tearDown()
    }

    func testLoadProfile() async throws {
        await viewModel.loadProfile()
        XCTAssertFalse(viewModel.userName.isEmpty)
    }
}
```

### UI Tests

```swift
import XCTest

final class MyAppUITests: XCTestCase {
    func testLoginFlow() throws {
        let app = XCUIApplication()
        app.launch()

        let emailField = app.textFields["emailField"]
        emailField.tap()
        emailField.typeText("test@example.com")

        let passwordField = app.secureTextFields["passwordField"]
        passwordField.tap()
        passwordField.typeText("password123")

        app.buttons["loginButton"].tap()

        XCTAssertTrue(app.navigationBars["Home"].exists)
    }
}
```

---

## Security & Privacy

### Keychain Storage

```swift
import Security

class KeychainManager {
    static func save(key: String, data: Data) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecValueData as String: data
        ]

        SecItemDelete(query as CFDictionary)
        let status = SecItemAdd(query as CFDictionary, nil)

        guard status == errSecSuccess else {
            throw KeychainError.unableToSave
        }
    }

    static func retrieve(key: String) throws -> Data {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecReturnData as String: true
        ]

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        guard status == errSecSuccess, let data = result as? Data else {
            throw KeychainError.itemNotFound
        }

        return data
    }
}

enum KeychainError: Error {
    case unableToSave
    case itemNotFound
}
```

### Privacy Permissions

```xml
<!-- Info.plist -->
<key>NSCameraUsageDescription</key>
<string>We need camera access to scan QR codes</string>

<key>NSLocationWhenInUseUsageDescription</key>
<string>We need your location to show nearby places</string>

<key>NSPhotoLibraryUsageDescription</key>
<string>We need access to save photos</string>
```

---

## Performance Optimization

### Instruments

- **Time Profiler:** CPU usage, hot paths
- **Allocations:** Memory usage, leaks
- **Leaks:** Retain cycles
- **Network:** Request timing, payload sizes

### Common Optimizations

```swift
// Image caching
import Kingfisher

KFImage(URL(string: imageURL))
    .placeholder { ProgressView() }
    .cacheMemoryOnly()
    .fade(duration: 0.25)

// Background processing
Task.detached(priority: .background) {
    // Heavy work
    await MainActor.run {
        // Update UI
    }
}

// Lazy initialization
private lazy var expensiveObject: ExpensiveObject = {
    ExpensiveObject()
}()
```

---

## Distribution

### TestFlight (Beta)

1. Archive build in Xcode
2. Upload to App Store Connect
3. Add external testers (up to 10,000)
4. Collect feedback via TestFlight app

### App Store

```bash
# Increment build number
agvtool next-version -all

# Create archive
xcodebuild archive -scheme MyApp -archivePath build/MyApp.xcarchive

# Export IPA
xcodebuild -exportArchive -archivePath build/MyApp.xcarchive -exportPath build -exportOptionsPlist ExportOptions.plist
```

---

## External Resources

- **Apple Developer:** https://developer.apple.com/documentation/
- **Swift.org:** https://swift.org/documentation/
- **SwiftUI Lab:** https://swiftui-lab.com/
- **Hacking with Swift:** https://www.hackingwithswift.com/
- **Point-Free (Advanced):** https://www.pointfree.co/

---

## Common Gotchas

1. **State Management:** Avoid `@State` for complex objects (use `@StateObject`)
2. **Memory Leaks:** Watch for retain cycles in closures (use `[weak self]`)
3. **Main Thread:** Always update UI on `@MainActor`
4. **Force Unwrapping:** Avoid `!` operator (use `if let` or guard)
5. **Asset Catalogs:** Always use vector PDFs for icons (support all sizes)
