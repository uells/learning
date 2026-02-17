## Установка ключа ed25519 для подключения по ssh к Серверу с Windows
Для начала мне необходимо добавить ключ для подключения к серверу на Windows 10, поскольку ключ у меня есть только в arch.
Для этого воспользуемся командой создания ключа 
```powershell
mkdir $env:USERPROFILE\.ssh -Force | Out-Null
ssh-keygen -t ed25519 -a 64 -f $env:USERPROFILE\.ssh\id_ed25519_win -C "win"
```

- `$env:USERPROFILE` - переменная окружения с путем к папке профиля в Windows текущего пользователя
- `-Force ` - не ругаться, если папка уже существует, при необходимости создать путь “как надо”
- `| Out-Null` - Без вывода. Команда выполнится "тихо"

- `-t` - тип ключа (ed25519, rsa и др.)
- `-a` - параметр усиления (KDF rounds) для шифрования приватного ключа паролем.
  - если установить passphrase (пароль) на приватный ключ, ssh-keygen шифрует приватный ключ
  - чтобы усложнить перебор пароля (bruteforce), используется key derivation function (KDF), и `-a` задаёт количество раундов.
  - больше -a → медленнее проверять один пароль → сложнее перебрать пароль.
- `-f` - задаёт путь и имя файла для ключа
- `-C` задаёт комментарий для публичного ключа

Далее выведем ключ в консоль.

```powershell
Get-Content $env:USERPROFILE\.ssh\id_ed25519_win.pub
```
- Get-Content (алиас: gc, cat, type) - командлет PowerShell для чтения содержимого файла/файлов построчно.

Скопируем содержимое публичного ключа и перенесем в arch для подключения к серверу и последующей устанвки публичного ключа на сервере.
На сервере вставляем ключ в файл `authorized_keys`.
```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys 
```
Не забываем установить необходимые права.

```bash
chmod 600 ~/.ssh/authorized_keys
```

Далее проверяем вход с windows.

```powershell
ssh -i $env:USERPROFILE\.ssh\id_ed25519_win user@server
```

Чтобы не прописывать каждый раз путь к ключу, пропишем конфиг в файле `C:\Users\<Username>\.ssh\config`

```sshconfig
Host myserver
  HostName server_ip
  User user
  IdentityFile ~/.ssh/id_ed25519_win
  IdentitiesOnly yes
```
- `User` - имя пользователя на удаленном сервере
- `IdentitiesOnly` - при `yes` SSH будет использовать только те ключи, которые явно указаны

После этого подключаемся короткой командой
```powershell
ssh myserver
```


