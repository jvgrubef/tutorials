---

# Configuração de PXE para Windows com Netboot.xyz

Este guia ensina como configurar um servidor PXE com Netboot.xyz para instalação do Windows. Ele inclui etapas para configurar um ambiente Docker, servidor Samba, e criação de imagens Windows PE.

---

## Pré-requisitos

1. **Usuário dedicado**: Crie um usuário para o compartilhamento via Samba e obtenha o `UID` e `GID` usando o comando:
   ```bash
   id <nome_do_usuario>
   ```

2. **Docker**: Certifique-se de que o Docker está instalado.

---

## Passo 1: Configuração do Docker

1. Crie um arquivo `docker-compose.yml` com o conteúdo abaixo, substituindo `<uid>` e `<gid>` pelos valores do usuário criado e ajustando o caminho para a pasta do Netboot.xyz.

   ```yaml
   version: '3'
   services:
     netbootxyz:
       image: lscr.io/linuxserver/netbootxyz:latest
       container_name: netbootxyz
       environment:
         - PUID=<uid>
         - PGID=<gid>
         - TZ=Etc/UTC
         - MENU_VERSION=1.9.9
       ports:
         - 3000:3000
         - 69:69/udp
         - 8080:8080
       volumes:
         - /path/to/netbootxyz/config:/config
         - /path/to/netbootxyz/assets:/assets
         - /path/to/netbootxyz/windows:/windows
       restart: unless-stopped
   ```

2. Execute o comando abaixo para iniciar o serviço:

   ```bash
   docker-compose up -d
   ```

---

## Passo 2: Configuração do Servidor NGINX

1. Edite o arquivo `default` em `config/nginx/site-confs`:

   ```nginx
   server {
       listen 8080;
       location /boot {
           alias /config/menus/;
           autoindex on;
       }
       location /windows {
           alias /windows;
           autoindex on;
       }
       location / {
           root /assets;
           autoindex on;
       }
   }
   ```

---

## Passo 3: Configuração do Samba

1. Instale o Samba (Exemplo para Ubuntu/Debian):
   ```bash
   apt install samba
   ```

2. Edite `/etc/samba/smb.conf` como mostrado abaixo, substituindo `<usuario>` pelo usuário dedicado e o caminho para a pasta de arquivos Windows:

   ```ini
   [global]
   acl allow execute always = True
   map to guest = Bad User
   guest account = <usuario>
   workgroup = WORKGROUP
   log level = 5
   unix extensions = No
   client min protocol = SMB2
   client max protocol = SMB3

   [PXE]
   path = /path/to/netbootxyz/windows
   readonly = yes
   browsable = yes
   guest ok = yes
   ```

---

## Passo 4: Criação do Windows PE

### Requisitos
- Uma máquina com Windows (pode ser uma VM).
- Baixe e instale o [Windows ADK](https://docs.microsoft.com/en-us/windows-hardware/get-started/adk-install) e o complemento do Windows PE.

### Comandos
1. **Crie a imagem do WinPE**:
   ```powershell
   copype amd64 C:\WinPE_amd64
   ```

2. **Monte a imagem WinPE**:
   Verifique o nome da imagem, provavelmente será `Microsoft Windows PE (amd64)` e passe no parametro `/Name`
   ```powershell
   Dism /Get-ImageInfo /ImageFile:"C:\WinPE_amd64\media\sources\boot.wim"
   Dism /Mount-Image /ImageFile:"C:\WinPE_amd64\media\sources\boot.wim" /Name:"Microsoft Windows PE (amd64)" /MountDir:C:\WinPE_amd64\mount
   ```

3. **Configuração adicional (idioma, pacotes)**:
   ```powershell
   Dism /Image:C:\WinPE_amd64\mount /Set-InputLocale:pt-br
   ```
   Se seu sistema for x86, remova ` (x86)` da linha de comando
   ```powershell
   Dism /Image:C:\WinPE_amd64\mount /Add-Package /PackagePath:"C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Windows Preinstallation Environment\amd64\WinPE_OCs\WinPE-WMI.cab"
   Dism /Image:C:\WinPE_amd64\mount /Add-Package /PackagePath:"C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Windows Preinstallation Environment\amd64\WinPE_OCs\WinPE-SecureStartup.cab"
   Dism /Image:C:\WinPE_amd64\mount /Add-Package /PackagePath:"C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Windows Preinstallation Environment\amd64\WinPE_OCs\PT-br\WinPE-SecureStartup_PT-br.cab"
   ```

4. **Salve e desmonte a imagem**:
   ```powershell
   Dism /Unmount-Image /MountDir:"C:\WinPE_amd64\mount" /commit
   ```

5. **Crie a ISO**:
   ```powershell
   MakeWinPEMedia /ISO C:\WinPE_amd64 C:\WinPE_amd64\WinPE_amd64.iso
   ```

---

## Passo 5: Configuração de Arquivos para o Netboot.xyz

1. **Estrutura de Pastas para Windows no Container**:
   - `/windows/WinPE/x64` → Arquivos extraidos da iso do WinPE
   - `/windows/Windows_10(x64)` → Arquivos extraidos da iso do Windows 10 (x64)
   - `/windows/Windows_11(x64)` → Arquivos extraidos da iso do Windows 11 (x64)

2. **Configuração de `boot.cfg` no dashboard do Netboot.xyz**:
   Altere os parâmetros para:
   ```bash
   set boot_domain http://<ip_do_servidor>:8080/boot
   set win_base_url http://<ip_do_servidor>:8080/windows/WinPE/
   ```

3. **Configuração do `windows.ipxe`**:
   Após `kernel http://${boot_domain}/wimboot` adicione:
   ```bash
   initrd -n setup.bat    http://${boot_domain}/setup.bat           setup.bat ||
   initrd -n winpeshl.ini http://${boot_domain}/winpeshl.ini        winpeshl.ini ||
   ```

---

## Passo 6: Criação de `setup.bat` e `winpeshl.ini`

Clique em `menu`, na guia para selecionar ou editar, crie os arquivos:

1. **Conteúdo do `winpeshl.ini`**:
   ```ini
   [LaunchApps]
   "setup.bat"
   ```

2. **Conteúdo do `setup.bat`**:
   ```batch
   @echo off
   title SK PXE AUX
   setlocal EnableDelayedExpansion
   chcp 65001
   cls
                                                                          
   set "target_dir=Z:"

   if "%1"=="" (
      wpeinit > nul
      net use %target_dir% /delete /y >nul 2>&1
      endlocal > nul
      start %~f0 "i" > nul
      echo.

      echo * Não feche ou clique em qualquer tecla dentro desta janela para manter o instalador em execução. 
      echo * Do contrario a maquina será reiniciada. O script extra é feito por causa do programa NET, que para ^(re^)montagem
      echo * precisa ser executado em um novo script
      echo.
      echo * Como funciona?
      echo * Você precisa ter um servidor Samba, e dentro da pasta compartilhada, devem ter as Iso's descompactadas
      echo * em pastas separadas, este script vai montar a pasta selecionada em Z:, e então dentro dessa unidade
      echo * vai procurar o executável setup.exe para iniciar a instalação
      echo.
      echo * Exemplo:
      echo.
      echo ^> \\endereço-do-servidor.local\
      echo     └─ windows-files
      echo        └─ windows_10^(x64^)
      echo        ^|   └─ setup.exe e o restante dos arquivos da Iso descompactada.
      echo        └─ windows_11^(x64^)
      echo            └─ setup.exe e a mesma coisa do de cima.

      pause >nul
      goto end
   )

   title SK PXE AUX - Installer
   set "server_standard=\\10.0.100.8\PXE\"
   set "server=%server_standard%"
   set "username="
   set "password="
   set "errorMenu="
   set "errorMsg="
   set "nextStep="
   set "setServer= (deixe em branco para não alterar)"
   :storageSetup
   cls

   echo Configuração de armazenamento:
   echo.

   if "%server_standard%"=="" (
      set "setServer="
      goto netData
   )

   echo Atual:
   echo Servidor: %server_standard%

   if not "%username%"=="" (
      echo Usuário: %username%
   ) else (
      echo Usuário definido: Não
   )

   if not "%password%"=="" (
      echo Senha definida: Sim
   ) else (
      echo Senha definida: Não
   )

   echo.

   set /p customChoice="Deseja customizar o caminho do servidor, usuário e senha? (S/N): "
   if /i "%customChoice%"=="S" (
      goto netData
   ) else if /i "%customChoice%"=="N" (
      goto netSetup
   ) else (
      echo Opção inválida
      goto storageSetup
   )

   :netData
   set /p server="Digite o caminho do servidor no formato: \\X.X.X.X\pasta\%setServer%: "

   if "%server%"=="" (
      if "%setServer%"=="" (
         cls
         echo Configuração de armazenamento:
         echo.
         echo É necessário definir o servidor
         goto netData
      )
      set "server=%server_standard%"
   ) else (
      set "server_standard=%server%"
   )

   set /p username="Digite o nome de usuário (deixe em branco se não precisar): "

   if "%username%"=="" (
      set "password="
      goto netSetup
   )

   set /p password="Digite a senha (deixe em branco se não precisar): "

   :netSetup
   cls

   echo A nova unidade %target_dir% está sendo montada.
   echo Buscando arquivos em %server%, aguarde.
   echo.

   if not "%username%"=="" (
      echo Usuário: %username%

      if not "%password%"=="" (
         echo Senha definida: Sim
      ) else (
         echo Senha definida: Não
      )

      net use %target_dir% %server% /user:%username% %password%
   ) else (
      echo Usuário definido: Não
      echo Senha definida: Não

      net use %target_dir% %server%
   )

   if errorlevel 1 (
      set "errorMsg=Erro ao montar a pasta %server% na unidade %target_dir%. Verifique a conexão de rede ou as permissões de acesso."
      set "nextStep=storageSetup"
      goto errorHandling
   )

   cls

   :menu
   if not "%errorMenu%"=="" (
      cls
      echo %errorMenu%
      echo.
      set "errorMenu="
   )

   cd /d "%target_dir%" || (
      set "errorMsg=O diretório não existe ou não está acessível."
      set "nextStep=menu"
      goto errorHandling
   )

   set count=0

   echo Selecionar uma pasta:
   echo -----------------------
   for /d %%D in (*) do (
      if /i not "%%D"=="WinPE" (
         set /a count+=1
         echo └─ !count! - %%D
         set "folder[!count!]=%%D"
      )
   )

   if %count% equ 0 (
      set "errorMsg=Nenhuma pasta encontrada."
      set "nextStep=menu"
      goto errorHandling
   )

   echo.
   echo Opções:
   echo -----------------------
   echo └─ S - Sair
   echo └─ R - Recarregar lista
   echo └─ V - Voltar a configuração de servidor
   echo.
   set /p choiceInstaller="Escolha uma pasta (1-%count%) ou uma das opções acima: "

   if /i "%choiceInstaller%"=="S" (
      goto end
   )
   if /i "%choiceInstaller%"=="R" (
      cls
      goto menu
   )
   if /i "%choiceInstaller%"=="V" (
      goto reset
   )

   if "!folder[%choiceInstaller%]!"=="" (
      set "errorMenu=Opção inválida! Por favor, escolha uma pasta válida."
      goto menu
   )

   cls
   set "selected_folder=!folder[%choiceInstaller%]!"

   if exist "!selected_folder!\setup.exe" (
      echo Você selecionou !selected_folder!, iniciando a instalação
      call "!selected_folder!\setup.exe"
   ) else (
      set "errorMsg=Não foi encontrado o instalador (setup.exe) na pasta !selected_folder!."
      set "nextStep=menu"
      goto errorHandling
   )

   goto menu

   :errorHandling
   echo %errorMsg%
   set /p choiceError="Deseja tentar novamente ou sair? (S/N): "

   if /i "%choiceError%"=="S" (
      goto %nextStep%
   ) else if /i "%choiceError%"=="N" (
      goto end
   ) else (
      echo Opção inválida.
      goto errorHandling
   )

   :reset
   net use %target_dir% /delete /y
   endlocal
   start %~f0 "i" >nul
   cls
   echo Nos vemos em breve.
   exit

   :end
   echo Saindo, a máquina será reiniciada em breve. Bye bye!
   echo %date% - %time%
   endlocal
   wpeutil reboot
   ```

---

## Passo 7: Reinicie os Serviços

Reinicie os serviços Docker e Samba:

```bash
docker restart netbootxyz
systemctl restart smbd
```

---

## Passo 8: Configuração do Servidor PXE no Roteador

Configure o roteador para apontar para o ip servidor PXE no arquivo de inicialização:
- Para sistemas EFI, `netboot.xyz.efi`.
- Para sistemas legados, `netboot.xyz.kpxe`.

---

Agora o ambiente PXE está pronto. Ao iniciar uma máquina na rede no modo PXE, acesse o menu de instalação do Windows via Netboot.xyz e siga as instruções para iniciar a instalação.
