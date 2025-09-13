> Usuario eliminado.
#### Herramientas:
* [ADRestore.exe](https://learn.microsoft.com/en-us/sysinternals/downloads/adrestore)
* [TheITBros](https://theitbros.com/restore-deleted-active-directory-user/)

<img width="900" height="500" alt="Usuarioelimminado" src="https://github.com/user-attachments/assets/ef75dd45-2378-4d44-8cd7-7037bec916bc" />

Para el caso que se muestra a continuacion, consideraremos que obtuvimos un archivo excel protegido con una contrasena, la cual se ha descifrado, resultando en:

| User       | Job Title                      | Permissions             | Notes                                                                 |
| ---------- | ------------------------------ | ----------------------- | --------------------------------------------------------------------- |
| Todd.Wolfe | Second-Line Support Technician | Remote Management Users | Leaver. Password was reset to NightT1meP1dg3on14 and account deleted. |
-------------------------------------------------------------------------------------------------------------------------------------------------

Cada vez que un usuario dentro de AD es eliminado puede ser reestablecido siempre y cuando se encuentre dentro del periodo de tiempo establecido dentro de su configuracion (180 dias por defecto). Durante este periodo de tiempo, los usuario eiminados son colocados dentro del contenedor **Deleted Objects** y su atributo **isDeleted** es configurado a **True**. El periodo de retencion se encuentra configurado por el atributo **tombstoneLifetime (TSL)**. Comencemos:

1) Revisaremos el valor configurado en el atributo **TSL** utilizando **powershell**.

```powershell
Get-ADRootDSE | Select DefaultNamingContext

# Salida Esperada:

# defaultNamingContext
# --------------------
# DC=voleur,DC=htb    
```

2) Sustituiremos el resultado obtenido en el siguiente comando.

```powershell
Get-ADObject -Identity "cn=Directory Service,cn=Windows NT, cn=Services,cn=Configuration,dc=voleur,dc=htb" -Properties tombstonelifetime

# Salida esperada:

#sDistinguishedName : cn=Directory Service,cn=Windows NT, cn=Services,cn=Configuration,dc=voleur,dc=htb
# Name              : Directory Service
# ObjectClass       : nTDSService
# ObjectGUID        : f0982ac9-1e60-4292-a68e-5f4fb0249703
# tombstonelifetime : 180

```

3) Utilizaremos **adrestore64.exe** para listar todos los objectos eliminados dentro **AD**. El resultado del siguiente comando muestra: **Organizationall Units (OUs), usuarios, computadoras, grupos, etc**. La segunda opcion consiste en realizar la busqueda de manera manual por objetos especificos como nuestro usaurio **Todd.Wolfe**.

```powershell
.\adrestore.exe "Todd Wolfe"

# Salida esperada:

# <SNIP>
# cn: Todd Wolfe
# DEL:1c6b1deb-c372-4cbb-87b1-15031de169db
# distinguishedName: CN=Todd Wolfe\0ADEL:1c6b1deb-<SNIP>-15031de169db,CN=Deleted Objects,DC=voleur,DC=htb
# lastKnownParent: OU=Second-Line Support Technicians,DC=voleur,DC=htb

# Found 1 item matching search criteria.
```

*Nota: Si es la primera vez que se ejecuta el binario **adrestore64.exe**, no olvidar agregar el parametro **-accepteula***.

4) Guardaremos el valor del atrubuto **GUID o DEL**, es necesario para reestablecer el usuario.

```powershell
.\adrestore64.exe -r 1c6b1deb-c372-4cbb-87b1-15031de169db

# Salida esperada:

# <SNIP>
# cn: Todd Wolfe
# DEL:1c6b1deb-c372-4cbb-87b1-15031de169db
# distinguishedName: CN=Todd Wolfe\0ADEL:1c6b1deb-<SNIP>-15031de169db,CN=Deleted Objects,DC=voleur,DC=htb
# lastKnownParent: OU=Second-Line Support Technicians,DC=voleur,DC=htb

Do you want to restore this object (y/n)? y

# Restore succeeded.

# Found 1 item matching search criteria.

```

5) Confirmaremos que el usuario **todd.wolfe** has sido reestablecido con exito.

```powershell
Get-ADUser -Identity 1c6b1deb-c372-4cbb-87b1-15031de169db

# Salida esperada:

# DistinguishedName : CN=Todd Wolfe,OU=Second-Line Support Technicians,DC=voleur,DC=htb
# Enabled           : True
# GivenName         : Todd
# Name              : Todd Wolfe
# ObjectClass       : user
# ObjectGUID        : 1c6b1deb-c372-4cbb-87b1-15031de169db
# SamAccountName    : todd.wolfe
# SID               : S-1-5-21-3927696377-1337352550-2781715495-1110
# Surname           : Wolfe
# UserPrincipalName : todd.wolfe@voleur.htb
```

Suponiendo que conocemos la contrasena del usuario **Todd Wolfe** y que hemos llegado a este punto, debe de ser posible para nosotros el poder utiizar herramientas como **Runas.exe** o **Invoke-RunasCs.ps1** en busca de lograr una escalacion de privilegios.

<img width="1100" height="500" alt="UsuarioReestablecido" src="https://github.com/user-attachments/assets/2ec70ff3-68b4-44bf-bef5-2712fe17cff1" />
