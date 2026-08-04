# React Native — введение для React-разработчиков

## Содержание

1. [Что такое React Native](#что-такое-react-native)
2. [Как работает React Native](#как-работает-react-native)
3. [React Native vs React](#react-native-vs-react)
4. [Новая архитектура (Fabric + TurboModules)](#новая-архитектура-fabric--turbomodules)
5. [Основные компоненты](#основные-компоненты)
6. [Стилизация](#стилизация)
7. [Навигация](#навигация)
8. [Состояние и данные](#состояние-и-данные)
9. [Нативные модули](#нативные-модули)
10. [Expo vs Bare CLI](#expo-vs-bare-cli)
11. [Отладка](#отладка)
12. [Публикация в сторах](#публикация-в-сторах)
13. [Когда НЕ выбирать React Native](#когда-не-выбирать-react-native)

---

## Что такое React Native

**React Native** — фреймворк для создания нативных мобильных приложений (iOS и Android) с использованием React. Код пишется на JavaScript/TypeScript, но рендеринг происходит в нативные UI-компоненты платформы — не WebView, не кросс-платформенный HTML/CSS.

```
React (Web) → ReactDOM → HTML / CSS / DOM
React Native → Native Renderer → UIKit (iOS) / Android Views
```

Ключевые особенности:
- **Один код** для iOS и Android (с нативными исключениями)
- **Нативные компоненты** — не WebView, настоящие `UIView`, `android.widget`
- **Hot Reload** — мгновенное обновление без пересборки
- **Нативные модули** — доступ к камере, GPS, push-уведомлениям, Bluetooth
- **Expo** — managed workflow без необходимости Xcode / Android Studio

> 💡 В 2026 году React Native используется в production в приложениях Meta (Facebook, Instagram), Shopify, Discord, Coinbase, Microsoft Office Mobile и тысячах других.

---

## Как работает React Native

### Архитектура (старая — Bridge)

```
┌──────────────────┐          ┌──────────────────┐
│  JavaScript      │          │  Native          │
│  Thread          │          │  Thread          │
│                  │  Bridge  │                  │
│  React           │◄────────►│  UIKit / Views   │
│  Components      │  (JSON)  │  Native Modules  │
│                  │          │                  │
└──────────────────┘          └──────────────────┘
```

JavaScript-поток вычисляет дерево компонентов, сериализует его в JSON и отправляет через асинхронный «мост» (Bridge) в нативный поток. Нативный поток десериализует и рендерит реальные UI-компоненты.

**Проблема:** Bridge — узкое место. Большие объёмы данных (скролл, жесты, анимации) вызывают задержки.

### Архитектура (новая — JSI)

```
┌──────────────────┐          ┌──────────────────┐
│  JavaScript      │          │  Native          │
│  (Hermes)        │◄────────►│  (Fabric +       │
│                  │   JSI    │   TurboModules)  │
│  React           │ (C++)    │                  │
│  Components      │          │                  │
└──────────────────┘          └──────────────────┘
```

**JSI (JavaScript Interface)** — C++ интерфейс, позволяющий JavaScript напрямую вызывать нативные методы без сериализации. Это убирает bottleneck Bridge и открывает путь к синхронным вызовам.

---

## React Native vs React

| | React (Web) | React Native |
|---|---|---|
| Рендеринг | HTML + CSS + DOM | Нативные UI-компоненты |
| Стилизация | CSS / CSS-in-JS / Tailwind | StyleSheet (подмножество CSS) |
| Навигация | React Router | React Navigation / Expo Router |
| События | `onClick`, `onChange` | `onPress`, `onChangeText` |
| Анимации | CSS transitions / Framer Motion | `Animated`, `react-native-reanimated` |
| Списки | `<ul>` / `map()` | `<FlatList>`, `<SectionList>` |
| Формы | `<form>`, `<input>` | `<TextInput>`, библиотеки форм |
| Отладка | Chrome DevTools | Flipper, React DevTools, Metro |
| Деплой | Vercel / Netlify | App Store / Google Play |
| Hot Reload | Vite HMR | Fast Refresh |

### Общие концепции

Всё, что вы знаете из React, работает:
- Компоненты, JSX, пропсы, состояние
- Хуки (`useState`, `useEffect`, `useRef`, `useMemo`, `useCallback`)
- Context API
- Custom Hooks
- Zustand, Jotai, TanStack Query
- TypeScript

### Ключевые различия

```jsx
// React (Web)
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);
  return (
    <div>
      <h1>Count: {count}</h1>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
    </div>
  );
}

// React Native
import { useState } from "react";
import { View, Text, Button, StyleSheet } from "react-native";

function Counter() {
  const [count, setCount] = useState(0);
  return (
    <View style={styles.container}>
      <Text style={styles.title}>Count: {count}</Text>
      <Button title="Increment" onPress={() => setCount(c => c + 1)} />
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, justifyContent: "center", alignItems: "center" },
  title: { fontSize: 24, fontWeight: "bold" },
});
```

---

## Новая архитектура (Fabric + TurboModules)

React Native постепенно переходит на новую архитектуру:

### Fabric (новый рендерер)

- Синхронный рендеринг (приоритетные обновления, как Concurrent React)
- Прямое взаимодействие с нативным UI через JSI
- Поддержка Suspense и concurrent features

### TurboModules (нативные модули)

- Ленивая загрузка нативных модулей (не все при старте)
- Прямые вызовы через JSI (без Bridge)
- Типобезопасность через CodeGen (автоматическая генерация типов)

### CodeGen

```json
// NativeSpec.json
{
  "nativeModules": {
    "Geolocation": {
      "methods": {
        "getCurrentPosition": {
          "promise": true,
          "params": []
        }
      }
    }
  }
}
```

CodeGen генерирует типобезопасные обёртки для JavaScript и нативного кода.

---

## Основные компоненты

### View и Text

```jsx
import { View, Text, StyleSheet } from "react-native";

function Card({ title, description }) {
  return (
    <View style={styles.card}>
      <Text style={styles.title}>{title}</Text>
      <Text style={styles.description}>{description}</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  card: { padding: 16, borderRadius: 8, backgroundColor: "#fff" },
  title: { fontSize: 18, fontWeight: "bold" },
  description: { fontSize: 14, color: "#666", marginTop: 4 },
});
```

### TouchableOpacity и Button

```jsx
import { TouchableOpacity, Button, Alert } from "react-native";

function Actions() {
  return (
    <View>
      <TouchableOpacity onPress={() => Alert.alert("Tapped!")}>
        <Text>Custom Button</Text>
      </TouchableOpacity>

      <Button title="Default Button" onPress={() => {}} />
    </View>
  );
}
```

### TextInput

```jsx
function SearchBar() {
  const [query, setQuery] = useState("");

  return (
    <TextInput
      style={styles.input}
      placeholder="Search..."
      value={query}
      onChangeText={setQuery}
      autoCapitalize="none"
      autoCorrect={false}
    />
  );
}
```

### FlatList (виртуализированный список)

```jsx
function UserList({ users }) {
  return (
    <FlatList
      data={users}
      keyExtractor={(item) => item.id}
      renderItem={({ item }) => (
        <View style={styles.row}>
          <Text>{item.name}</Text>
          <Text>{item.email}</Text>
        </View>
      )}
      refreshing={isRefreshing}
      onRefresh={handleRefresh}
      onEndReached={handleLoadMore}
      onEndReachedThreshold={0.5}
    />
  );
}
```

### ScrollView

```jsx
function FormScreen() {
  return (
    <ScrollView contentContainerStyle={styles.container}>
      <TextInput placeholder="Name" />
      <TextInput placeholder="Email" keyboardType="email-address" />
      <Button title="Submit" onPress={handleSubmit} />
    </ScrollView>
  );
}
```

### Image

```jsx
<Image
  source={{ uri: "https://example.com/image.jpg" }}
  style={{ width: 200, height: 200 }}
  resizeMode="cover"
/>

// Локальное изображение
<Image source={require("./assets/logo.png")} style={{ width: 100, height: 100 }} />
```

### SafeAreaView и StatusBar

```jsx
import { SafeAreaView, StatusBar } from "react-native";

function App() {
  return (
    <SafeAreaView style={styles.container}>
      <StatusBar barStyle="dark-content" />
      <MainContent />
    </SafeAreaView>
  );
}
```

---

## Стилизация

### StyleSheet

```jsx
import { StyleSheet } from "react-native";

const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 16,
    backgroundColor: "#f5f5f5",
  },
  card: {
    padding: 16,
    borderRadius: 8,
    backgroundColor: "#fff",
    shadowColor: "#000",
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    elevation: 3,
  },
});
```

### Flexbox (отличия от CSS)

React Native использует Flexbox, но с некоторыми отличиями:

- `flexDirection` по умолчанию — `column` (не `row` как в CSS)
- Нет `display: grid`, `float`, `position: relative`
- `position` поддерживает только `relative` и `absolute`
- Нет shorthand-свойств (`margin: 10px 20px` не работает)

```jsx
<View style={{
  flex: 1,
  flexDirection: "row",
  justifyContent: "space-between",
  alignItems: "center",
}}>
  <View style={{ width: 50, height: 50, backgroundColor: "red" }} />
  <View style={{ width: 50, height: 50, backgroundColor: "blue" }} />
</View>
```

### Responsive design

```jsx
import { Dimensions, useWindowDimensions } from "react-native";

function ResponsiveLayout() {
  const { width, height } = useWindowDimensions();
  const isPortrait = height > width;

  return (
    <View style={{
      flexDirection: isPortrait ? "column" : "row",
      flex: 1,
    }}>
      <Sidebar />
      <Content />
    </View>
  );
}
```

### Styled Components в React Native

```jsx
import styled from "styled-components/native";

const Container = styled.View`
  flex: 1;
  justify-content: center;
  align-items: center;
  background-color: #f5f5f5;
`;

const Title = styled.Text`
  font-size: 24px;
  font-weight: bold;
  color: #333;
`;

function Screen() {
  return (
    <Container>
      <Title>Hello, React Native!</Title>
    </Container>
  );
}
```

### NativeWind (Tailwind для React Native)

```jsx
import { View, Text } from "react-native";

function Card() {
  return (
    <View className="p-4 bg-white rounded-lg shadow-md">
      <Text className="text-lg font-bold text-gray-800">Title</Text>
      <Text className="text-sm text-gray-500 mt-1">Description</Text>
    </View>
  );
}
```

---

## Навигация

### Expo Router (рекомендуемый)

Expo Router — файловая маршрутизация, аналог Next.js App Router:

```
app/
├── _layout.tsx       # корневой layout
├── index.tsx         # главная страница (/)
├── (tabs)/
│   ├── _layout.tsx   # tab layout
│   ├── home.tsx      # вкладка "Home"
│   └── profile.tsx   # вкладка "Profile"
├── (auth)/
│   ├── login.tsx     # /login
│   └── register.tsx  # /register
└── settings/
    └── index.tsx     # /settings
```

```tsx
// app/_layout.tsx
import { Stack } from "expo-router";

export default function RootLayout() {
  return (
    <Stack>
      <Stack.Screen name="(tabs)" options={{ headerShown: false }} />
      <Stack.Screen name="(auth)" options={{ headerShown: false }} />
    </Stack>
  );
}

// app/(tabs)/_layout.tsx
import { Tabs } from "expo-router";

export default function TabLayout() {
  return (
    <Tabs>
      <Tabs.Screen name="home" options={{ title: "Home", tabBarIcon: HomeIcon }} />
      <Tabs.Screen name="profile" options={{ title: "Profile", tabBarIcon: ProfileIcon }} />
    </Tabs>
  );
}
```

### React Navigation

```jsx
import { NavigationContainer } from "@react-navigation/native";
import { createNativeStackNavigator } from "@react-navigation/native-stack";

const Stack = createNativeStackNavigator();

function App() {
  return (
    <NavigationContainer>
      <Stack.Navigator>
        <Stack.Screen name="Home" component={HomeScreen} />
        <Stack.Screen name="Details" component={DetailsScreen} />
      </Stack.Navigator>
    </NavigationContainer>
  );
}

function HomeScreen({ navigation }) {
  return (
    <View>
      <Text>Home Screen</Text>
      <Button title="Go to Details" onPress={() => navigation.navigate("Details")} />
    </View>
  );
}
```

---

## Состояние и данные

### Локальное состояние

```jsx
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);
  return <Button title={`Count: ${count}`} onPress={() => setCount(c => c + 1)} />;
}
```

### AsyncStorage

```jsx
import AsyncStorage from "@react-native-async-storage/async-storage";

async function saveUser(user) {
  await AsyncStorage.setItem("user", JSON.stringify(user));
}

async function loadUser() {
  const json = await AsyncStorage.getItem("user");
  return json ? JSON.parse(json) : null;
}
```

### Zustand (работает как в React)

```jsx
import { create } from "zustand";

const useAuthStore = create((set) => ({
  user: null,
  login: (user) => set({ user }),
  logout: () => set({ user: null }),
}));

function Profile() {
  const { user, logout } = useAuthStore();
  return (
    <View>
      <Text>{user?.name}</Text>
      <Button title="Logout" onPress={logout} />
    </View>
  );
}
```

### TanStack Query (работает как в React)

```jsx
import { useQuery } from "@tanstack/react-query";

function UserList() {
  const { data, isLoading } = useQuery({
    queryKey: ["users"],
    queryFn: () => fetch("https://api.example.com/users").then((r) => r.json()),
  });

  if (isLoading) return <ActivityIndicator />;

  return (
    <FlatList
      data={data}
      keyExtractor={(item) => item.id}
      renderItem={({ item }) => <Text>{item.name}</Text>}
    />
  );
}
```

---

## Нативные модули

### Доступ к камере

```jsx
import * as ImagePicker from "expo-image-picker";

async function pickImage() {
  const permission = await ImagePicker.requestMediaLibraryPermissionsAsync();
  if (!permission.granted) return;

  const result = await ImagePicker.launchImageLibraryAsync({
    mediaTypes: ["images"],
    quality: 0.8,
  });

  if (!result.canceled) {
    console.log("Selected:", result.assets[0].uri);
  }
}
```

### Push-уведомления

```jsx
import * as Notifications from "expo-notifications";

async function registerForPushNotifications() {
  const { status } = await Notifications.requestPermissionsAsync();
  if (status !== "granted") return;

  const token = await Notifications.getExpoPushTokenAsync({
    projectId: "your-project-id",
  });

  console.log("Push token:", token.data);
}

Notifications.addNotificationReceivedListener((notification) => {
  console.log("Notification received:", notification.request.content.title);
});
```

### Геолокация

```jsx
import * as Location from "expo-location";

async function getCurrentLocation() {
  const { status } = await Location.requestForegroundPermissionsAsync();
  if (status !== "granted") return;

  const location = await Location.getCurrentPositionAsync({});
  return {
    latitude: location.coords.latitude,
    longitude: location.coords.longitude,
  };
}
```

---

## Expo vs Bare CLI

### Expo (рекомендуется)

```bash
npx create-expo-app my-app
cd my-app
npx expo start
```

**Плюсы:**
- Не нужен Xcode / Android Studio для разработки
- EAS Build — облачная сборка для iOS/Android
- EAS Update — OTA-обновления без ревью сторов
- Готовые модули (камера, геолокация, уведомления)
- Expo Router — файловая маршрутизация

**Минусы:**
- Ограничения при использовании кастомных нативных модулей
- Expo Dev Client нужен для нативного кода

### Bare CLI (React Native CLI)

```bash
npx @react-native-community/cli init my-app
```

**Плюсы:**
- Полный контроль над нативным кодом
- Нет ограничений Expo

**Минусы:**
- Нужен Xcode (macOS) для iOS
- Нужен Android Studio для Android
- Ручная настройка нативных модулей

> 💡 В 2026 году **Expo — рекомендуемый способ** начать React Native проект. Даже если нужен нативный код, Expo C API и Config Plugins позволяют добавлять нативные модули без «eject».

---

## Отладка

### React DevTools

```bash
npx react-devtools
```

Инспектирование дерева компонентов, пропсов, состояния — как в браузере.

### Flipper

Нативная отладка:
- Просмотр нативного UI-дерева
- Сетевые запросы
- База данных (SQLite)
- Логи

### Console.log и Metro

```bash
npx expo start --dev-client
```

Metro bundler показывает логи в терминале.

### Debugging в VS Code

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Expo App",
      "type": "exponent",
      "request": "launch"
    }
  ]
}
```

---

## Публикация в сторах

### EAS Build (Expo)

```bash
npm install -g eas-cli

eas login
eas build:configure

# iOS
eas build --platform ios --profile production

# Android
eas build --platform android --profile production
```

### eas.json

```json
{
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal"
    },
    "preview": {
      "distribution": "internal"
    },
    "production": {
      "autoIncrement": true
    }
  },
  "submit": {
    "production": {
      "ios": {
        "appleId": "your@email.com",
        "ascAppId": "123456789"
      },
      "android": {
        "serviceAccountKeyPath": "./google-services.json",
        "track": "production"
      }
    }
  }
}
```

### EAS Update (OTA)

```bash
eas update --branch production --message "Fix login bug"
```

Обновления JavaScript-кода без ревью App Store / Google Play.

---

## Когда НЕ выбирать React Native

React Native не подходит, если:

- **Тяжёлые анимации / игры** — используйте Unity, Flutter или нативные фреймворки
- **AR / VR** — нативные SDK (ARKit, ARCore) дают больше контроля
- **Bluetooth / IoT** — нативные модули существуют, но отладка сложнее
- **Критическая производительность** — нативный код быстрее (хотя новая архитектура缩小ла разрыв)
- **Одно приложение** — если не планируете iOS + Android, нативная разработка проще
