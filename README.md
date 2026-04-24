# Tesorería Logia Lucero del Alba

Esta app usa Firebase para autenticación multiusuario, multi-logia y almacenamiento en la nube.

## Qué se cambió
- Autenticación con Firebase Auth (email/contraseña).
- Almacenamiento en Firestore (datos separados por logia).
- Usuario admin (cguzman@propertyplus.cl) puede gestionar logias y usuarios.
- Usuarios normales ven solo sus logias asignadas, con permisos de viewer o editor.
- Acceso desde cualquier dispositivo.

## Configuración necesaria
1. Crea un proyecto en Firebase.
2. Habilita `Authentication > Email/Password`.
3. Crea una base de datos Firestore.
4. Actualiza reglas de Firestore (ver abajo).
5. En `index.html`, reemplaza `FIREBASE_CONFIG` por tus valores.

## Firebase config sample
```js
const FIREBASE_CONFIG = {
  apiKey: '...',
  authDomain: '...firebaseapp.com',
  projectId: '...',
  storageBucket: '...appspot.com',
  messagingSenderId: '...',
  appId: '...'
};
```

## Reglas de Firestore
Ve a Firebase Console > Firestore > Rules y pega:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    function isAdmin() {
      return request.auth.token.email == 'cguzman@propertyplus.cl'
          || request.auth.token.role == 'admin'
          || (exists(/databases/$(database)/documents/users/$(request.auth.uid))
              && get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == 'admin');
    }

    function hasLogiaAccess(logia) {
      return (request.auth.token.logias != null && logia in request.auth.token.logias)
          || (exists(/databases/$(database)/documents/users/$(request.auth.uid))
              && logia in get(/databases/$(database)/documents/users/$(request.auth.uid)).data.logias);
    }

    function canWriteLogia(logia) {
      return (request.auth.token.role == 'editor' && request.auth.token.logias != null && logia in request.auth.token.logias)
          || (exists(/databases/$(database)/documents/users/$(request.auth.uid))
              && get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == 'editor'
              && logia in get(/databases/$(database)/documents/users/$(request.auth.uid)).data.logias);
    }

    match /apps/{logia} {
      allow read: if request.auth != null && (isAdmin() || hasLogiaAccess(logia));
      allow write: if request.auth != null && (isAdmin() || canWriteLogia(logia));
    }

    match /users/{uid} {
      allow read: if request.auth != null && (request.auth.uid == uid || isAdmin());
      allow write: if request.auth != null && isAdmin();
    }
  }
}
```

## Reglas de Storage (para logos)
Ve a Firebase Console > Storage > Rules y pega:

```
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /logos/{logia}.png {
      allow read: if request.auth != null && 
        (request.auth.token.role == 'admin' || logia in request.auth.token.logias);
      allow write: if request.auth != null && request.auth.token.role == 'admin';
    }
  }
}
```

## Uso
- Abre `index.html` en el navegador.
- Admin puede cambiar logos por logia en el menú "Gestionar Logias".
- Inicia sesión.
- Si eres admin, verás menú "Admin" para gestionar logias/usuarios.
- Para otras logias, crea documentos en Firestore y asigna claims a usuarios.

## Claims de usuarios
En Firebase Console > Authentication > Users > selecciona usuario > Custom claims:

- Admin: `{"role": "admin"}`
- Editor multilogía: `{"logias": ["logia1", "logia2"], "role": "editor"}`
- Viewer una logia: `{"logias": ["logia1"], "role": "viewer"}`
