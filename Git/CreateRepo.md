# Создание репозитория на GitHub
## Из терминала (в VS-Code, например)
1. Инициализировать git
```powershell
git init
```
2. Добавить файлы
```powershell
git add .
```
3. Первый коммит
```powershell
git commit -m "initial project structure"
```
4. Создание ветки `dev`
```powershell
git branch dev
git checkout dev

# Или
git checkout -b dev
```
5. Проверить текущую ветку
```powershell
git branch
```
6. Дальше необходимо создать репозиторий на GitHub и скопировать URL
7. Выполнить команду 
```powershell
git remote add origin https://github.com/USERNAME/repository.git
```
8. Запушить ветку `dev`
```powershell
git push -u origin dev
```
9. Чтобы наследоваться от ветки dev при создании других веток нужно перейти на ветку `dev`
```powershell
# Переходим
git checkout dev
# Создаем новую ветку
git checkout -b feature/cache-component
```
10. Работаем. Коммитим. Пушим изменения
```powershell
git push -u origin feature/cache-component
```