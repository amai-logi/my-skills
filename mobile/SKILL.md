---
name: mobile
description: "Build mobile applications for iOS and Android. Covers Flutter (Dart, Material 3, Provider/Riverpod, Hive), React Native (Expo, TypeScript, Zustand), and native Kotlin/Swift when hardware access or platform-specific features are required. Handles secure token storage, offline support, push notifications, navigation, AI provider integration, and design language adaptation. Use this skill when the user asks to build a mobile app, add a mobile client to an existing backend, or build a client-side-only app with AI features."
---

# Mobile Skill

## When to Skip

- Web-only application — go to `frontend`
- Apps Script project — go to `google-apps-script`
- The architecture doesn't include a mobile client

## When to Use

- The architecture specifies a mobile app (iOS, Android, or both)
- Adding a mobile client to an existing backend
- Building a device-specific app (barcode scanner, NFC reader)
- Client-side-only apps with AI integration, voice/audio features, or local-first storage


Build mobile applications that connect to backends produced by the `backend` skill, or run entirely client-side. Default to React Native for cross-platform when sharing a TypeScript web codebase. Use Flutter for cross-platform when the app is mobile-first or doesn't share code with a web frontend. Use native Kotlin (Android) or Swift (iOS) only when the use case demands it.

## Step 0: Read Prerequisites

Before starting:
1. Read `design-language.md` — adapt the Logitech design language for mobile (see Step 4)
2. Read `guardrails.md` — all cross-cutting rules apply
3. Check if the `software-architecture` or `backend` skill has already produced an API contract — the mobile app consumes it

## Step 1: Evaluate the Approach

### Cross-Platform vs Native

| Option | Default? | When to use |
|--------|----------|-------------|
| **React Native** | Yes (web+mobile) | Most apps with a web frontend — forms, lists, dashboards, CRUD, chat. Shares TypeScript with web frontend. Single codebase for iOS + Android. |
| **Flutter** | Yes (mobile-first) | Mobile-first apps, client-side-only apps, apps with rich animations or custom UI, when there's no web frontend to share code with. Single Dart codebase for iOS + Android + desktop + web. |
| **Native Kotlin (Android)** | No | Hardware-intensive: barcode/QR scanners, NFC readers, Bluetooth LE, camera with custom processing, background location tracking. |
| **Native Swift (iOS)** | No | Same hardware needs on iOS. Or when the app is iOS-only. |
| **Both native** | No | When both platforms need hardware access that cross-platform frameworks can't bridge reliably. |

**Decision rule:** If the project has a React/TypeScript web frontend, use React Native to share code. If the app is mobile-first or standalone (no web frontend), use Flutter. Go native only when core features require hardware access that cross-platform can't handle.

### When cross-platform is NOT enough

| Feature | React Native | Flutter | Native needed? |
|---------|-------------|---------|----------------|
| Camera (photos, video) | Works via expo-camera | Works via `camera` package | No |
| Barcode/QR scanning | Works via expo-barcode-scanner | Works via `mobile_scanner` | No, unless high-speed industrial scanning |
| NFC reading/writing | Limited library support | Works via `nfc_manager` | Only for complex NFC flows |
| Bluetooth LE | Works via react-native-ble-plx | Works via `flutter_blue_plus` | No, unless complex pairing flows |
| Background location | Limited on iOS | Works via `geolocator` + `workmanager` | Yes, for continuous tracking |
| Custom camera processing (ML on frames) | Possible but slow | Works via `google_mlkit_*` | Yes, for real-time frame processing |
| Push notifications | Works via expo-notifications | Works via `firebase_messaging` | No |
| Offline-first with sync | Works with WatermelonDB / MMKV | Works with Hive / Isar / drift | No |
| Biometric auth | Works via expo-local-authentication | Works via `local_auth` | No |
| Audio recording | Works via expo-av | Works via `record` package | No |
| Text-to-speech | Limited | Works via `flutter_tts` | No |
| Speech recognition | Limited | Works via `speech_to_text` | No |

## Step 2: Project Structure

### React Native (Expo)

Use Expo with the managed workflow. Eject only if a native module requires it.

```
mobile-app/
├── app/                        # Expo Router file-based routing
│   ├── (tabs)/                 # Tab navigator
│   │   ├── _layout.tsx
│   │   ├── index.tsx           # Home tab
│   │   └── profile.tsx         # Profile tab
│   ├── (auth)/                 # Auth flow (unauthenticated)
│   │   ├── _layout.tsx
│   │   ├── login.tsx
│   │   └── callback.tsx        # OAuth callback handler
│   ├── _layout.tsx             # Root layout (auth check)
│   └── [resource]/
│       ├── index.tsx           # List view
│       └── [id].tsx            # Detail view
├── src/
│   ├── api/
│   │   └── client.ts           # API client (typed, from OpenAPI spec)
│   ├── auth/
│   │   ├── provider.tsx        # Auth context provider
│   │   ├── storage.ts          # Secure token storage
│   │   └── pkce.ts             # PKCE flow implementation
│   ├── components/
│   │   ├── ui/                 # Design language components
│   │   └── shared/             # App-specific shared components
│   ├── hooks/
│   │   └── useApi.ts           # Data fetching hook
│   ├── styles/
│   │   └── tokens.ts           # Design language tokens for RN
│   └── utils/
├── app.json                    # Expo config
├── package.json
├── tsconfig.json
└── eas.json                    # EAS Build config
```

### Flutter

Use Flutter with Material 3. Dart-only — no native platform code unless a plugin requires it.

```
flutter-app/
├── lib/
│   ├── main.dart                    # App entry, Provider/Riverpod setup, theme
│   ├── models/                      # Plain Dart data classes (JSON serialization)
│   │   └── {resource}.dart
│   ├── providers/                   # State management (ChangeNotifier or Riverpod)
│   │   └── app_state.dart
│   ├── services/                    # Business logic, API clients, storage
│   │   ├── api_service.dart         # HTTP client (http or dio)
│   │   ├── auth_service.dart        # Auth (google_sign_in, flutter_secure_storage)
│   │   ├── local_storage_service.dart  # On-device persistence (Hive, SharedPreferences)
│   │   └── {feature}_service.dart
│   ├── screens/                     # Full-page widgets
│   │   ├── home_screen.dart
│   │   ├── settings_screen.dart
│   │   └── {feature}_screen.dart
│   └── widgets/                     # Reusable UI components
│       └── {component}.dart
├── test/                            # Unit and widget tests
│   ├── {service}_test.dart
│   └── {widget}_test.dart
├── android/                         # Android-specific config
├── ios/                             # iOS-specific config
├── pubspec.yaml                     # Dependencies and assets
├── analysis_options.yaml            # Lint rules
└── CLAUDE.md                        # Project context for AI agents
```

### Native Kotlin (Android)

```
android-app/
├── app/
│   └── src/
│       └── main/
│           ├── java/com/project/
│           │   ├── MainActivity.kt
│           │   ├── ui/
│           │   │   ├── screens/        # Compose screens
│           │   │   ├── components/     # Reusable composables
│           │   │   └── theme/          # Design language theme
│           │   ├── data/
│           │   │   ├── api/            # Retrofit API client
│           │   │   ├── repository/     # Data repositories
│           │   │   └── local/          # Room database (offline)
│           │   ├── auth/
│           │   │   └── AuthManager.kt  # PKCE + token management
│           │   └── scanner/            # Hardware-specific (barcode, NFC)
│           ├── AndroidManifest.xml
│           └── res/
├── build.gradle.kts
└── gradle/
```

### Native Swift (iOS)

```
ios-app/
├── App/
│   ├── ContentView.swift
│   ├── Screens/                # SwiftUI views
│   ├── Components/             # Reusable views
│   ├── Theme/                  # Design language tokens
│   ├── API/                    # URLSession API client
│   ├── Auth/                   # ASWebAuthenticationSession + Keychain
│   ├── Data/                   # Core Data or SwiftData (offline)
│   └── Scanner/                # AVCaptureSession (barcode/QR)
├── Info.plist
└── Package.swift
```

## Step 3: Authentication

Mobile auth uses **OAuth 2.0 Authorization Code + PKCE**. The mobile app is a public client — no client secret.

### React Native — PKCE Flow

```typescript
// src/auth/pkce.ts
import * as AuthSession from 'expo-auth-session';
import * as SecureStore from 'expo-secure-store';

const discovery = {
  authorizationEndpoint: 'https://accounts.google.com/o/oauth2/v2/auth',
  tokenEndpoint: 'https://oauth2.googleapis.com/token',
};

export function useAuth() {
  const [request, response, promptAsync] = AuthSession.useAuthRequest(
    {
      clientId: process.env.EXPO_PUBLIC_OAUTH_CLIENT_ID!,
      scopes: ['openid', 'profile', 'email'],
      redirectUri: AuthSession.makeRedirectUri(),
      usePKCE: true,
    },
    discovery
  );

  useEffect(() => {
    if (response?.type === 'success') {
      const { code } = response.params;
      exchangeCodeForTokens(code, request!.codeVerifier!);
    }
  }, [response]);

  return { login: () => promptAsync(), request };
}

async function exchangeCodeForTokens(code: string, codeVerifier: string) {
  const tokenResponse = await AuthSession.exchangeCodeAsync(
    {
      clientId: process.env.EXPO_PUBLIC_OAUTH_CLIENT_ID!,
      code,
      redirectUri: AuthSession.makeRedirectUri(),
      extraParams: { code_verifier: codeVerifier },
    },
    discovery
  );

  // Store tokens securely — NEVER in AsyncStorage
  await SecureStore.setItemAsync('access_token', tokenResponse.accessToken);
  await SecureStore.setItemAsync('refresh_token', tokenResponse.refreshToken!);
}
```

### Flutter — Google Sign-In

```dart
// lib/services/auth_service.dart
import 'package:google_sign_in/google_sign_in.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

class AuthService {
  static final _googleSignIn = GoogleSignIn(
    scopes: ['email', 'profile'],
  );
  static const _storage = FlutterSecureStorage();

  static Future<GoogleSignInAccount?> signIn() async {
    final account = await _googleSignIn.signIn();
    if (account != null) {
      final auth = await account.authentication;
      await _storage.write(key: 'access_token', value: auth.accessToken);
    }
    return account;
  }

  static Future<void> signOut() async {
    await _googleSignIn.signOut();
    await _storage.deleteAll();
  }

  /// For API key-based services (e.g., Gemini API), store the key securely
  static Future<void> saveApiKey(String key) async {
    await _storage.write(key: 'api_key', value: key);
  }

  static Future<String?> getApiKey() async {
    return await _storage.read(key: 'api_key');
  }
}
```

### Flutter — AI Provider Auth (API Key + Service Account)

For apps that call AI APIs directly (no custom backend), Flutter supports multiple auth patterns:

```dart
// lib/services/ai_provider.dart
abstract class AIProvider {
  Future<String> processText(String input);
  Future<String> processAudio(List<int> audioBytes);
}

// API Key mode — simplest, user provides key in setup wizard
class GeminiStudioProvider implements AIProvider {
  final String apiKey;
  final String model;

  GeminiStudioProvider({required this.apiKey, required this.model});

  @override
  Future<String> processText(String input) async {
    final response = await http.post(
      Uri.parse('https://generativelanguage.googleapis.com/v1beta/models/$model:generateContent?key=$apiKey'),
      headers: {'Content-Type': 'application/json'},
      body: jsonEncode({'contents': [{'parts': [{'text': input}]}]}),
    );
    return _parseResponse(response);
  }
}

// Service account mode — user imports JSON credentials
class GeminiVertexProvider implements AIProvider {
  final ServiceAccountCredentials credentials;
  final String projectId;
  final String region;
  final String model;

  // Uses googleapis_auth to create authenticated client
}
```

### Token Storage

| Platform | Secure storage | Insecure (never use) |
|----------|---------------|---------------------|
| React Native | `expo-secure-store` (Keychain on iOS, EncryptedSharedPreferences on Android) | AsyncStorage, MMKV for tokens |
| Flutter | `flutter_secure_storage` (Keychain on iOS, EncryptedSharedPreferences on Android) | SharedPreferences, Hive for tokens |
| Android native | `EncryptedSharedPreferences` or Android Keystore | SharedPreferences, files |
| iOS native | Keychain Services (`kSecClassGenericPassword`) | UserDefaults, files |

### Token Refresh

```typescript
// src/api/client.ts
async function authenticatedFetch(url: string, options: RequestInit = {}) {
  let token = await SecureStore.getItemAsync('access_token');

  let response = await fetch(url, {
    ...options,
    headers: { ...options.headers, Authorization: `Bearer ${token}` },
  });

  if (response.status === 401) {
    token = await refreshAccessToken();
    response = await fetch(url, {
      ...options,
      headers: { ...options.headers, Authorization: `Bearer ${token}` },
    });
  }

  return response;
}

async function refreshAccessToken(): Promise<string> {
  const refreshToken = await SecureStore.getItemAsync('refresh_token');
  if (!refreshToken) throw new Error('No refresh token — re-login required');

  const response = await fetch(discovery.tokenEndpoint, {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type: 'refresh_token',
      client_id: process.env.EXPO_PUBLIC_OAUTH_CLIENT_ID!,
      refresh_token: refreshToken,
    }).toString(),
  });

  const tokens = await response.json();
  await SecureStore.setItemAsync('access_token', tokens.access_token);
  if (tokens.refresh_token) {
    await SecureStore.setItemAsync('refresh_token', tokens.refresh_token);
  }

  return tokens.access_token;
}
```

## Step 4: Design Language Adaptation

The Logitech design language (`design-language.md`) is designed for web. Mobile needs adaptation:

### Token Mapping (React Native)

```typescript
// src/styles/tokens.ts
export const colors = {
  bgPage: '#f3f1ec',        // --bg-page
  card: '#ffffff',            // --card
  border: '#d7d7d7',         // --border
  textPrimary: '#1b1b1b',   // --text-primary
  textSecondary: '#8f8f8f', // --text-secondary
  textMuted: '#a0a0a0',     // --text-muted
  logiTeal: '#00fdcf',      // --logi-teal (CTA only)
  logiDarkGreen: '#0d3a38', // --logi-dark-green
  lightGreen: '#d6f5f0',    // --brand-light-green
  error: '#E8344E',         // --red
};

export const typography = {
  fontFamily: 'Inter',       // Load via expo-font or bundled
  h1: { fontSize: 28, fontWeight: '800' as const, letterSpacing: -1 },
  h2: { fontSize: 18, fontWeight: '700' as const, letterSpacing: -0.3 },
  body: { fontSize: 15, fontWeight: '400' as const, lineHeight: 24 },
  label: { fontSize: 14, fontWeight: '600' as const },
  small: { fontSize: 12, fontWeight: '500' as const },
};

export const spacing = {
  xs: 4,
  sm: 8,
  md: 16,
  lg: 24,
  xl: 32,
};

export const radius = {
  sm: 8,
  md: 12,
  lg: 16,
  pill: 100,
};
```

### Token Mapping (Flutter)

```dart
// lib/theme.dart
import 'package:flutter/material.dart';

class AppTheme {
  static ThemeData get light => ThemeData(
    useMaterial3: true,
    colorSchemeSeed: const Color(0xFF673AB7), // deep purple
    scaffoldBackgroundColor: const Color(0xFFF5F5F5),
    cardTheme: const CardThemeData(
      elevation: 0,
      shape: RoundedRectangleBorder(
        borderRadius: BorderRadius.all(Radius.circular(16)),
        side: BorderSide(color: Color(0xFFE0E0E0)),
      ),
    ),
    inputDecorationTheme: InputDecorationTheme(
      border: OutlineInputBorder(borderRadius: BorderRadius.circular(12)),
      contentPadding: const EdgeInsets.symmetric(horizontal: 16, vertical: 14),
    ),
  );

  static ThemeData get dark => ThemeData(
    useMaterial3: true,
    brightness: Brightness.dark,
    colorSchemeSeed: const Color(0xFF673AB7),
  );
}
```

### Mobile-Specific Adaptations

| Web design language | Mobile adaptation |
|--------------------|-------------------|
| `rem` units | Points/dp — convert `1rem` ≈ `16dp` |
| Max-width `860px` / `1060px` | Full screen width with horizontal padding `16dp` |
| Hover states | Press states (opacity or scale feedback) |
| Cursor pointer | Touch target minimum `44x44dp` (Apple HIG) / `48x48dp` (Material) |
| Modal with backdrop blur | Bottom sheet or full-screen modal |
| Toast at bottom center | SnackBar (Flutter) or system-native toast |
| Table layout | Card list or expandable rows |
| Two-column form | Single column (always) |
| Sidebar | Drawer navigation or bottom tabs |

### Component Patterns (React Native)

```typescript
// Button — pill shape, same as web
import { Pressable, Text, StyleSheet } from 'react-native';

function PrimaryButton({ title, onPress, disabled }: Props) {
  return (
    <Pressable
      onPress={onPress}
      disabled={disabled}
      style={({ pressed }) => [
        styles.button,
        pressed && styles.pressed,
        disabled && styles.disabled,
      ]}
    >
      <Text style={styles.text}>{title}</Text>
    </Pressable>
  );
}

const styles = StyleSheet.create({
  button: {
    backgroundColor: colors.logiTeal,
    borderRadius: radius.pill,
    paddingVertical: 14,
    paddingHorizontal: 28,
    alignItems: 'center',
  },
  pressed: { opacity: 0.9, transform: [{ scale: 0.98 }] },
  disabled: { opacity: 0.5 },
  text: {
    color: colors.logiDarkGreen,
    fontSize: typography.label.fontSize,
    fontWeight: '700',
  },
});
```

## Step 4b: Flutter State Management

### Provider (ChangeNotifier) — default for simple apps

```dart
// lib/providers/app_state.dart
class AppState extends ChangeNotifier {
  List<NoteFile> _files = [];
  bool _isLoading = false;

  List<NoteFile> get files => _files;
  bool get isLoading => _isLoading;

  Future<void> loadFiles() async {
    _isLoading = true;
    notifyListeners();
    _files = await _storageService.getFiles();
    _isLoading = false;
    notifyListeners();
  }
}

// main.dart
void main() {
  runApp(
    ChangeNotifierProvider(
      create: (_) => AppState(),
      child: const MyApp(),
    ),
  );
}
```

### When to split state

| Signal | Action |
|--------|--------|
| ChangeNotifier > 300 lines | Split into feature-scoped providers (NotesProvider, SettingsProvider, AIConfigProvider) |
| Unnecessary rebuilds (changing settings rebuilds notes list) | Use `MultiProvider` + `Selector` to scope rebuilds |
| Complex async flows | Consider Riverpod for better async handling |

## Step 5: Offline Support

Evaluate whether the app needs offline support. Most apps don't — only add it when:
- Users work in areas with poor connectivity (warehouses, field work)
- Core functionality must work without a network (barcode scanning, data entry)
- Data must sync back when connectivity returns

### When needed — Offline Strategy

| Approach | React Native | Flutter |
|----------|-------------|---------|
| **Key-value storage** | MMKV | Hive (`hive_flutter`) |
| **Structured local DB** | WatermelonDB | drift (SQLite) or Isar |
| **Offline queue** | Custom with NetInfo | Custom with `connectivity_plus` |
| **Cache-first** | react-query/swr with persistence | Custom or cached_network_image |

### Flutter — Offline Queue Pattern

```dart
// Listen for connectivity changes, process queued items when online
class OfflineQueueService {
  final _connectivity = Connectivity();

  void startListening() {
    _connectivity.onConnectivityChanged.listen((result) {
      if (result != ConnectivityResult.none) {
        _processPendingItems();
      }
    });
  }

  Future<void> _processPendingItems() async {
    final pending = await _storage.getPendingItems();
    for (final item in pending) {
      try {
        await _apiService.submit(item);
        await _storage.removePending(item.id);
      } catch (e) {
        break; // retry next time
      }
    }
  }
}
```

### Conflict Resolution

When offline writes sync back, conflicts can occur. Strategy:
- **Last-write-wins** for simple data (user profiles, settings)
- **Server-wins** for inventory/financial data (canonical source is the server)
- **Merge with user prompt** for collaborative data (show diff, let user choose)

## Step 6: Push Notifications

### React Native (Expo)

```typescript
import * as Notifications from 'expo-notifications';
import * as Device from 'expo-device';

async function registerForPushNotifications() {
  if (!Device.isDevice) return null; // Simulators can't receive pushes

  const { status } = await Notifications.requestPermissionsAsync();
  if (status !== 'granted') return null;

  const token = await Notifications.getExpoPushTokenAsync({
    projectId: process.env.EXPO_PUBLIC_PROJECT_ID,
  });

  // Send token to backend for storage
  await api.post('/users/me/push-token', { token: token.data });

  return token;
}
```

### Flutter

```dart
import 'package:firebase_messaging/firebase_messaging.dart';

Future<void> setupPushNotifications() async {
  final messaging = FirebaseMessaging.instance;
  final settings = await messaging.requestPermission();
  if (settings.authorizationStatus == AuthorizationStatus.authorized) {
    final token = await messaging.getToken();
    await apiService.registerPushToken(token!);
  }

  // Handle foreground messages
  FirebaseMessaging.onMessage.listen((RemoteMessage message) {
    // Show local notification or update UI
  });
}
```

### Backend Integration

The backend stores push tokens and sends notifications via:
- **Expo Push API** for React Native (Expo handles APNs + FCM)
- **Firebase Cloud Messaging (FCM)** for Flutter and native Android
- **Apple Push Notification Service (APNs)** for native iOS

## Step 7: Navigation

### React Native — Expo Router (file-based)

```typescript
// app/_layout.tsx — Root layout with auth guard
import { Redirect, Stack } from 'expo-router';
import { useAuth } from '../src/auth/provider';

export default function RootLayout() {
  const { isAuthenticated, isLoading } = useAuth();

  if (isLoading) return <SplashScreen />;
  if (!isAuthenticated) return <Redirect href="/(auth)/login" />;

  return <Stack screenOptions={{ headerShown: false }} />;
}

// app/(tabs)/_layout.tsx — Tab navigation
import { Tabs } from 'expo-router';

export default function TabLayout() {
  return (
    <Tabs screenOptions={{
      tabBarActiveTintColor: colors.logiDarkGreen,
      tabBarInactiveTintColor: colors.textMuted,
      tabBarStyle: { backgroundColor: colors.card, borderTopColor: colors.border },
    }}>
      <Tabs.Screen name="index" options={{ title: 'Home', tabBarIcon: HomeIcon }} />
      <Tabs.Screen name="profile" options={{ title: 'Profile', tabBarIcon: ProfileIcon }} />
    </Tabs>
  );
}
```

### Flutter — Navigator 2.0 / GoRouter

```dart
// Use MaterialApp with named routes for simple apps
MaterialApp(
  initialRoute: '/',
  routes: {
    '/': (context) => const HomeScreen(),
    '/settings': (context) => const SettingsScreen(),
    '/note': (context) => const NoteDetailScreen(),
  },
);

// Use GoRouter for complex navigation (deep links, guards)
final router = GoRouter(
  redirect: (context, state) {
    final isLoggedIn = authService.isAuthenticated;
    if (!isLoggedIn && !state.matchedLocation.startsWith('/login')) {
      return '/login';
    }
    return null;
  },
  routes: [
    GoRoute(path: '/', builder: (_, __) => const HomeScreen()),
    GoRoute(path: '/note/:id', builder: (_, state) =>
      NoteDetailScreen(id: state.pathParameters['id']!)),
  ],
);
```

### Native Kotlin — Jetpack Compose Navigation

```kotlin
@Composable
fun AppNavigation(authState: AuthState) {
    val navController = rememberNavController()

    NavHost(navController, startDestination = if (authState.isAuthenticated) "home" else "login") {
        composable("login") { LoginScreen(navController) }
        composable("home") { HomeScreen(navController) }
        composable("scan") { ScannerScreen(navController) }
        composable("item/{id}") { backStackEntry ->
            ItemDetailScreen(itemId = backStackEntry.arguments?.getString("id")!!)
        }
    }
}
```

## Step 8: Build & Distribution

### React Native — EAS Build

```json
// eas.json
{
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal"
    },
    "preview": {
      "distribution": "internal",
      "ios": { "simulator": false }
    },
    "production": {
      "autoIncrement": true
    }
  },
  "submit": {
    "production": {
      "ios": { "appleId": "your@email.com", "ascAppId": "123456789" },
      "android": { "serviceAccountKeyPath": "./google-play-key.json" }
    }
  }
}
```

```bash
# Development build (with dev client)
eas build --profile development --platform all

# Preview build (internal testing)
eas build --profile preview --platform all

# Production build + submit to stores
eas build --profile production --platform all
eas submit --platform all
```

### Flutter — Build Commands

```bash
flutter pub get          # Install dependencies
flutter run              # Run on connected device / emulator
flutter test             # Run all tests
flutter analyze          # Static analysis / linting
flutter build apk        # Build Android APK (debug)
flutter build appbundle  # Build Android App Bundle (release, for Play Store)
flutter build ios        # Build iOS (requires Xcode)
```

For Play Store release:
1. Generate upload keystore: `keytool -genkey -v -keystore upload-keystore.jks -keyalg RSA -keysize 2048 -validity 10000`
2. Configure signing in `android/app/build.gradle.kts`
3. Build: `flutter build appbundle`
4. Upload to Play Console

### Native — CI/CD via GitHub Actions

```yaml
# Android
- run: ./gradlew assembleRelease
- uses: r0adkll/upload-google-play@v1
  with:
    serviceAccountJsonPlainText: ${{ secrets.GOOGLE_PLAY_SA }}
    packageName: com.project.app
    releaseFiles: app/build/outputs/apk/release/*.apk
    track: internal

# iOS
- run: xcodebuild archive -scheme App -archivePath App.xcarchive
- run: xcodebuild -exportArchive -archivePath App.xcarchive -exportPath export
- uses: apple-actions/upload-testflight-build@v1
```

## Step 9: Deliver

Present deliverables in this order:
1. **Approach evaluation** — React Native vs Flutter vs native, with justification
2. **Scaffolded project** — working project structure with auth, navigation, API client
3. **Design language tokens** — mobile adaptation of `design-language.md`
4. **Build config** — EAS, Flutter build, or native CI/CD setup
5. **README** — how to run locally, environment variables, build commands

List all files created at the end.

## Rules

**Cross-cutting guardrails:** Follow all rules in `guardrails.md` — security, cost, data integrity, process, resilience, and maintenance.

| Do | Don't |
|----|-------|
| Use `expo-secure-store` / `flutter_secure_storage` / Keychain / EncryptedSharedPreferences for tokens | Store tokens in AsyncStorage, SharedPreferences, Hive, or UserDefaults |
| Use PKCE for all OAuth flows | Use implicit flow or store client secrets in the app |
| Minimum touch target 44x44dp (iOS) / 48x48dp (Android) | Make tiny tap targets |
| Adapt design language for mobile (single column, touch states, dp units) | Copy web CSS values directly |
| Use Expo managed workflow unless a native module requires ejecting | Eject preemptively "just in case" |
| Use Flutter Material 3 with `colorSchemeSeed` for theming | Hard-code colors everywhere |
| Use Provider or Riverpod for Flutter state management | Use setState for global state |
| Test on real devices for hardware features | Only test on simulators |
| Handle token refresh transparently | Force users to re-login on token expiry |
| Design for poor connectivity (loading states, retry, timeout) | Assume the network is always available |
| Add offline support only when the use case demands it | Add offline-first complexity to every app |
| Use file-based routing (Expo Router) for React Native | Use manual React Navigation setup |
| Pin Expo SDK / Flutter SDK version | Use `latest` or mix SDK versions |
| Use `flutter analyze` before committing | Skip linting |

## Handoff

### Input
- API contract from `backend` skill (endpoints the app consumes)
- Architecture decisions from `software-architecture` skill (which clients, auth pattern)
- `design-language.md` for styling (adapted to mobile)

### Output
- Scaffolded mobile project (React Native, Flutter, or native)
- Auth implementation (PKCE + secure token storage, or API key management)
- API client (typed, from OpenAPI spec or direct API calls)
- Build configuration (EAS, Flutter build, or native CI/CD)

### Previous skills
- `software-architecture` — decided multi-client architecture requiring a mobile app
- `backend` — produced the API contract the mobile app consumes

### Next skills

| Next | What it receives |
|------|-----------------|
| `testing` | Mobile project to add component tests (React Native Testing Library), widget tests (Flutter), or UI tests (native) |
| `deployment` | Build config for CI/CD pipeline (EAS Build, Flutter build scripts, or native build scripts) |

## Continuous Improvement

### Skill Health Check

At the end of every use, evaluate and report if any of the following apply:

- **Skill gap:** "This project needs [X] and no current skill covers it. Consider creating a `[name]` skill." — e.g., a wearable/IoT skill, a tablet-optimized layout skill.
- **Skill split:** "This skill's [section] is growing complex enough to be its own skill." — e.g., if native Android and native iOS patterns diverge enough to warrant separate skills.
- **Skill overlap:** "This skill and `[other skill]` both cover [topic]. Consider consolidating." — e.g., if `frontend` skill's React knowledge overlaps significantly with React Native.

Only report when genuinely applicable. Don't force observations.

### Learnings

Corrections and refinements discovered during use. When the user overrides a recommendation or a default doesn't fit, record it here so future uses benefit.

_(Empty — will accumulate over time)_
