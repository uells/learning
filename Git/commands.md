# Git — краткий конспект команд

## Инициализация
`git init` — создать новый репозиторий

## Состояние и информация
`git status` — текущее состояние файлов  
`git log` — история коммитов  
`git log --oneline` — краткая история  
`git diff` — изменения в файлах  
`git diff --staged` — изменения в staging  

## Добавление и коммиты
`git add file` — добавить файл в staging  
`git add .` — добавить все изменения  
`git commit -m "сообщение"` — зафиксировать изменения  
`git commit --amend` — изменить последний коммит  

## Отмена изменений
`git restore file` — отменить изменения файла  
`git restore --staged file` — убрать файл из staging  

## Работа с ветками
`git branch` — список веток  
`git branch name` — создать ветку  
`git switch name` — перейти в ветку  
`git switch -c name` — создать и перейти  
`git merge name` — слить ветку  

## Удалённые репозитории
`git remote -v` — список remote  
`git remote add origin URL` — добавить remote  
`git push` — отправить изменения  
`git push -u origin main` — первый push  
`git pull` — получить и слить изменения  
`git fetch` — получить без слияния  

## Файлы и папки
`git rm file` — удалить файл  
`git mv old new` — переименовать / переместить  

## .gitignore
`git check-ignore -v file` — проверить, почему файл игнорируется  

## Типичный рабочий цикл
изменил файлы →  
`git status` →  
`git add .` →  
`git commit -m "описание"` →  
`git push`

