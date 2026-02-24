# Cómo publicar paquetes de BP con Github actions

Para subir la versión
```bash
# aumentar el tercer número:
npm version patch
# pasar de beta a rc:
npm version prerelease --preid=rc --force
# aumentar el cuarto número:
npm version prerelease
```