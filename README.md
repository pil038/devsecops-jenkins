# Звіт: Налаштування Jenkins CI/CD Pipeline

## Скріншоти
Всі скріншоти до роботи знаходяться у папці: [screenshots/](screenshots/)

---

## Мета
Налаштування Jenkins CI/CD pipeline з автоматичним деплоєм через SSH, інтеграцією з GitHub та Pipeline as Code з використанням Docker на Windows.

---

## Етап 1: Базове налаштування та перевірка Jenkins

**Крок 1.** Встановлено та запущено Docker Desktop. Перевірено працездатність командою `docker ps`.

**Крок 2.** Запущено Jenkins у Docker-контейнері на порту 8080:
```
docker run -d --name jenkins -p 8080:8080 -p 50000:50000 -v jenkins_home:/var/jenkins_home jenkins/jenkins:lts
```

**Крок 3.** Відкрито Jenkins у браузері за адресою `http://localhost:8080`. Введено початковий пароль отриманий командою:
```
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

**Крок 4.** Встановлено suggested plugins, створено адміністратора з іменем `admin`. Отримано доступ до головної сторінки Jenkins (версія 2.555.2).

**Крок 5.** Створено Freestyle job `devsecops_lab` з тестовою shell-командою:
```bash
echo "Test"
echo $BUILD_NUMBER
```

**Крок 6.** Запущено першу збірку. Консоль підтвердила успішне виконання: вивід `Test`, номер збірки `1`, статус `Finished: SUCCESS`.

---

## Етап 2: Підготовка до деплою по SSH

**Крок 7.** Запущено другий Docker-контейнер — веб-сервер з підтримкою SSH:
```
docker run -d --name webserver -p 80:80 rastasheep/ubuntu-sshd:18.04
```

**Крок 8.** На веб-сервер встановлено Nginx та налаштовано права доступу до папки `/var/www/html`:
```
docker exec webserver apt-get install -y nginx
docker exec webserver chmod -R 777 /var/www/html
```

**Крок 9.** Отримано IP-адресу веб-сервера (`172.17.0.3`) та перевірено що SSH працює (`sshd is running`).

**Крок 10.** Згенеровано SSH-ключову пару для Jenkins та скопійовано публічний ключ на веб-сервер. Перевірено SSH-з'єднання між контейнерами — отримано відповідь `SSH works!`.

**Крок 11.** У Jenkins встановлено плагін **Publish Over SSH** через **Manage Jenkins → Plugins → Available plugins**.

**Крок 12.** У глобальних налаштуваннях Jenkins (**Manage Jenkins → System**) додано SSH-сервер:
- Name: `webserver`
- Hostname: `172.17.0.3`
- Username: `root`
- Remote Directory: `/var/www/html`

Тест з'єднання показав **Success**.

---

## Етап 3: Налаштування та перевірка деплою

**Крок 13.** У налаштуваннях job'а `devsecops_lab` змінено команду збірки на генерацію HTML-файлу:
```bash
echo "<html><body><h1>Hello World</h1></body></html>" > index.html
```

**Крок 14.** У розділі **Послесборочные операции** додано крок **Send build artifacts over SSH**:
- Source files: `index.html`
- Exec command: `echo $BUILD_ID`

**Крок 15.** Запущено збірку #2. Консоль підтвердила успішний деплой: `SSH: Transferred 1 file(s)`, `Finished: SUCCESS`.

**Крок 16.** Відкрито браузер за адресою `http://localhost` — веб-сторінка з текстом **Hello World** успішно завантажилась.

---

## Етап 4: Інтеграція з GitHub

**Крок 17.** Створено новий публічний репозиторій `devsecops-jenkins` на GitHub.

**Крок 18.** У репозиторії створено файл `index.html` з вмістом:
```html
<html><body><h1>GitHub</h1></body></html>
```

**Крок 19.** Отримано публічний SSH-ключ Jenkins та додано його як Deploy Key у налаштуваннях репозиторію (**Settings → Deploy keys → Add deploy key**) з назвою `jenkins`.

**Крок 20.** Додано GitHub до списку відомих хостів Jenkins:
```
docker exec jenkins bash -c "ssh-keyscan github.com >> /var/jenkins_home/.ssh/known_hosts"
```

**Крок 21.** У налаштуваннях job'а `devsecops_lab` у розділі **Управление исходным кодом** вибрано Git та вказано:
- Repository URL: `git@github.com:pil038/devsecops-jenkins.git`
- Credentials: SSH-ключ Jenkins
- Branch: `*/main`

**Крок 22.** Запущено збірку #3. Консоль підтвердила успішне клонування репозиторію з GitHub та передачу файлу на веб-сервер:
- `Cloning repository git@github.com:pil038/devsecops-jenkins.git`
- `Commit message: "Create index.html"`
- `SSH: Transferred 1 file(s)`
- `Finished: SUCCESS`

---

## Етап 5: Pipeline as Code

**Крок 23.** Створено новий проект типу **Pipeline** з назвою `devsecops_pipeline`.

**Крок 24.** У розділі **Pipeline** написано декларативний Groovy скрипт з трьома етапами:
```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                echo 'Building..'
            }
        }
        stage('Test') {
            steps {
                echo 'Testing..'
            }
        }
        stage('Deploy') {
            steps {
                echo 'Deploying....'
            }
        }
    }
}
```

**Крок 25.** Запущено Pipeline #1. Консоль підтвердила успішне проходження всіх трьох етапів:
- `[Pipeline] { (Build)` → `Building..`
- `[Pipeline] { (Test)` → `Testing..`
- `[Pipeline] { (Deploy)` → `Deploying....`
- `Finished: SUCCESS`

---

## Висновок

Успішно налаштовано повний Jenkins CI/CD pipeline на базі Docker:

| Етап | Результат |
|---|---|
| Розгортання Jenkins у Docker | ✅ |
| Налаштування SSH деплою на веб-сервер | ✅ |
| Автоматична передача файлів по SSH | ✅ |
| Інтеграція з GitHub через Deploy Key | ✅ |
| Pipeline as Code (Build → Test → Deploy) | ✅ |

Використання Docker дозволило розгорнути повноцінну CI/CD інфраструктуру без необхідності налаштування окремих віртуальних машин. Jenkins автоматично клонує код з GitHub репозиторію та деплоїть його на веб-сервер через SSH при кожному запуску збірки.
