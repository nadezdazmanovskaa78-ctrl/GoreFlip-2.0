name: Build APK

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:

      - name: Скачать код репозитория
        uses: actions/checkout@v4

      - name: Установить Java 21
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'

      - name: Установить Gradle 8.7
        uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: '8.7'

      - name: Создать Android-проект
        run: |
          mkdir -p app/src/main/assets app/src/main/java/com/goreflip/game
          cp index.html app/src/main/assets/index.html
          cat > settings.gradle <<'EOF'
          pluginManagement {
              repositories { google(); mavenCentral(); gradlePluginPortal() }
          }
          dependencyResolutionManagement {
              repositories { google(); mavenCentral() }
          }
          rootProject.name = 'GoreFlip'
          include ':app'
          EOF
          cat > build.gradle <<'EOF'
          plugins {
              id 'com.android.application' version '8.5.2' apply false
          }
          EOF
          cat > app/build.gradle <<'EOF'
          plugins {
              id 'com.android.application'
          }
          android {
              namespace 'com.goreflip.game'
              compileSdk 34
              defaultConfig {
                  applicationId 'com.goreflip.game'
                  minSdk 21
                  targetSdk 34
                  versionCode 1
                  versionName '1.0'
              }
              compileOptions {
                  sourceCompatibility JavaVersion.VERSION_17
                  targetCompatibility JavaVersion.VERSION_17
              }
          }
          EOF
          cat > app/src/main/AndroidManifest.xml <<'EOF'
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">
              <uses-permission android:name="android.permission.INTERNET"/>
              <application android:label="GoreFlip" android:theme="@android:style/Theme.Material.NoActionBar">
                  <activity android:name=".MainActivity" android:exported="true" android:configChanges="orientation|screenSize">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN"/>
                          <category android:name="android.intent.category.LAUNCHER"/>
                      </intent-filter>
                  </activity>
              </application>
          </manifest>
          EOF
          cat > app/src/main/java/com/goreflip/game/MainActivity.java <<'EOF'
          package com.goreflip.game;

          import android.app.Activity;
          import android.os.Bundle;
          import android.webkit.WebSettings;
          import android.webkit.WebView;

          public class MainActivity extends Activity {
              @Override
              protected void onCreate(Bundle savedInstanceState) {
                  super.onCreate(savedInstanceState);
                  WebView webView = new WebView(this);
                  WebSettings settings = webView.getSettings();
                  settings.setJavaScriptEnabled(true);
                  settings.setDomStorageEnabled(true);
                  webView.loadUrl("file:///android_asset/index.html");
                  setContentView(webView);
              }
          }
          EOF
          SDKM=$ANDROID_HOME/cmdline-tools/latest/bin/sdkmanager
          yes | $SDKM --licenses > /dev/null || true
          $SDKM "platforms;android-34" "build-tools;34.0.0"

      - name: Собрать APK
        run: gradle assembleDebug

      - name: Сохранить APK
        uses: actions/upload-artifact@v4
        with:
          name: GoreFlip-apk
          path: app/build/outputs/apk/debug/app-debug.apk
