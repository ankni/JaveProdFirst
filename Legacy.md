name: Java_CI_with_Gradle

on:
  push:
    branches: ["main"] # Добавил чтобы еще фича бренч тригерил пайплайн
  pull_request:
    branches: ["main"]

jobs:

  test_job:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout repository
      uses: actions/checkout@v3

    - name: Set up JDK 11
      uses: actions/setup-java@v3
      with:
        java-version: '11'
        distribution: 'temurin'
                                            # Добавил кеширование для ускорения запуска
    - name: Grant execute permission for Gradle
      run: chmod +x gradlew

    - name: Run tests with Gradle
      run: ./gradlew test         # добавил флаг  --continue. он позволит не останавливать выполнение на первом же ошибочном тесте, а прогнать все тесты и показать полный список ошибок.

  build_job:
    runs-on: ubuntu-latest
    needs: test_job
    
    steps:
    - name: Clone repository
      uses: actions/checkout@v3

    - name: Set up JDK 11
      uses: actions/setup-java@v3
      with:
        java-version: '11'
        distribution: 'temurin'

    - name: Grant execute permission to Gradle
      run: chmod +x gradlew

    - name: Build with Gradle
      run: ./gradlew build --warning-mode all

    - name: Debug Check build output  # Явно указал воркдир, вроде как это полезно
      run: ls -la build/libs/

    - name: Rename JAR for Docker
      run: |
        ls -la build/libs/                           # Упростил переименование JAR
        JAR_FILE=$(ls build/libs/*.jar | head -n 1)
        mv "$JAR_FILE" build/libs/app.jar

    - name: Verify JAR before Docker
      run: ls -la build/libs/app.jar

    - name: Log in to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build and Push Docker Image
      uses: docker/build-push-action@v5
      with:
        context: .
        file: Dockerfile
        push: true
        tags: egorravino/demo-image-java-app:v1-main-${{ github.sha }}

