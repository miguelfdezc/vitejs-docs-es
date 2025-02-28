# Funcionalidades

En un nivel muy básico, desarrollar usando Vite no es muy diferente de usar un servidor de archivos estático. Sin embargo, Vite proporciona muchas mejoras sobre las importaciones de ESM nativas para admitir varias funciones que normalmente se ven en las configuraciones basadas en paquetes.

## Resolución de dependencias de NPM y preempaquetado

Las importaciones nativas de ES no admiten importaciones de módulos descubiertos como las siguientes:

```js
import { someMethod } from 'my-dep'
```

Lo anterior arrojará un error en el navegador. Vite detectará tales importaciones de módulos descubiertos en todos los archivos fuente servidos y realizará lo siguiente:

1. [Preempaquetado](./dep-pre-bundling) para mejorar la velocidad de carga de la página y convertir los módulos CommonJS/UMD a ESM. El paso previo al empaquetado se realiza con [esbuild](http://esbuild.github.io/) y hace que el tiempo de inicio en frío de Vite sea significativamente más rápido que cualquier empaquetador basado en JavaScript.

2. Vuelve a escribir las importaciones en direcciones URL válidas como `/node_modules/.vite/deps/my-dep.js?v=f3sf2ebd` para que el navegador pueda importarlas correctamente.

**Las dependencias están fuertemente almacenadas en caché**

Vite almacena en caché las solicitudes de dependencias a través de encabezados HTTP, por lo que si deseas editar/depurar localmente una dependencia, sigue los pasos [aquí](./dep-pre-bundling#cache-de-navegador).

## Hot Module Replacement

Vite proporciona una [API de HMR](./api-hmr) sobre ESM nativo. Los marcos de trabajo con capacidades HMR pueden aprovechar la API para proporcionar actualizaciones instantáneas y precisas sin recargar la página o eliminar el estado de la aplicación. Vite proporciona integraciones HMR propias para [Vue Single File Components](https://github.com/vitejs/vite/tree/main/packages/plugin-vue) y [React Fast Refresh](https://github.com/vitejs/vite/tree/main/packages/plugin-react). También hay integraciones oficiales para Preact a través de [@prefresh/vite](https://github.com/JoviDeCroock/prefresh/tree/main/packages/vite).

Ten en cuenta que no necesitas configurarlos manualmente: cuando [creas una aplicación a través de `create-vite`](./), las plantillas seleccionadas ya las tendrán preconfiguradas.

## TypeScript

Vite admite la importación de archivos `.ts` desde su primer uso.

Vite solo realiza la transpilación en archivos `.ts` y **NO** realiza verificación de tipos. Asume que la verificación de tipos está a cargo de tu IDE y el proceso de compilación (puede ejecutar `tsc --noEmit` en el script de compilación o instalar `vue-tsc` y ejecutar `vue-tsc --noEmit` para también verificar el tipo de su archivos `*.vue`).

Vite usa [esbuild](https://github.com/evanw/esbuild) para transpilar TypeScript en JavaScript, que es entre 20 y 30 veces más rápido que `tsc` puro, y las actualizaciones de HMR pueden reflejarse en el navegador en menos de 50 ms.

Usa la sintaxis de [importaciones y exportaciones de solo tipo](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-8.html#type-only-imports-and-export) para evitar problemas potenciales como las importaciones de solo tipo que se agrupan incorrectamente, por ejemplo:

```ts
import type { T } from 'only/types'
export type { T }
```

### Opciones del compilador de TypeScript

Algunos campos de configuración en `compilerOptions` en `tsconfig.json` requieren atención especial.

#### `isolatedModules`

Debes configurarse en `true`.

Esto se debe a que `esbuild` solo realiza la transpilación sin información de tipo, no admite ciertas características como const enum e importaciones implícitas de solo tipo.

Debes configurar `"isolatedModules": true` en el `tsconfig.json` en `compilerOptions`, para que TS te advierta sobre las funcionalidades omitidas con la transpilación aislada.

Sin embargo, algunas bibliotecas (por ejemplo, [`vue`](https://github.com/vuejs/core/issues/1228)) no funcionan bien con `"isolatedModules": true`. Puedes usar `"skipLibCheck": true` para suprimir temporalmente los errores hasta que se solucione.

#### `useDefineForClassFields`

A partir de Vite 2.5.0, el valor predeterminado será `true` si el destino de TypeScript es "ESNext". Esto es consistente con el [comportamiento de `tsc` 4.3.2 y versiones posteriores](https://github.com/microsoft/TypeScript/pull/42663). También es el comportamiento esperado en tiempo de ejecución de ECMAScript. Pero esto puede ser contrario para aquellos que provienen de otros lenguajes de programación o versiones anteriores de TypeScript. Puedes leer más sobre la transición en las [notas de la versión de TypeScript 3.7](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-7.html#the-usedefineforclassfields-flag-and-el-declare-property-modifier).

Si estás utilizando una biblioteca que depende en gran medida de los campos de clase, ten cuidado con el uso previsto de la biblioteca.

La mayoría de las bibliotecas esperan `"useDefineForClassFields": true`, como [MobX](https://mobx.js.org/installation.html#use-spec-obediente-transpilation-for-class-properties), [Vue Class Components 8.x](https://github.com/vuejs/vue-class-component/issues/465), etc. Pero algunas bibliotecas aún no han hecho la transición a este nuevo valor predeterminado, incluido [`lit-element`](https://github.com/lit/lit-element/issues/1030). Configura explícitamente `useDefineForClassFields` en `false` en estos casos.

#### Otras opciones del compilador que afectan el resultado de la compilación

- [`extends`](https://www.typescriptlang.org/tsconfig#extends)
- [`importsNotUsedAsValues`](https://www.typescriptlang.org/tsconfig#importsNotUsedAsValues)
- [`preserveValueImports`](https://www.typescriptlang.org/tsconfig#preserveValueImports)
- [`jsxFactory`](https://www.typescriptlang.org/tsconfig#jsxFactory)
- [`jsxFragmentFactory`](https://www.typescriptlang.org/tsconfig#jsxFragmentFactory)

Si migrar el codigo base a `"isolatedModules": true` es un esfuerzo arduo, es posible que puedas facilitar el trabajo con un complemento de terceros como [rollup-plugin-friendly-type-imports](https://www.npmjs.com/package/rollup-plugin-friendly-type-imports). Sin embargo, Vite no admite oficialmente este enfoque.

### Tipos de clientes

Los tipos predeterminados de Vite son para su API de Node.js. Para ajustar el entorno del código del lado del cliente en una aplicación Vite, agrega un archivo de declaración `d.ts`:

```typescript
/// <reference types="vite/client" />
```

Además, puedes agregar `vite/client` a `compilerOptions.types` de tu `tsconfig`:

```json
{
  "compilerOptions": {
    "types": ["vite/client"]
  }
}
```

Esto proporcionará los siguientes tipos de librerías:

- Importaciones de recursos (por ejemplo, importar un archivo `.svg`)
- Tipos para las [variables de entorno](./env-and-mode#variables-de-entorno) inyectadas por Vite en `import.meta.env`
- Tipos para la [API de HMR](./api-hmr) en `import.meta.hot`

:::tip
Para anular la escritura predeterminada, declárala antes de la referencia de triple slash. Por ejemplo, para hacer que la importación predeterminada de `*.svg` sea un componente de React:

```ts
declare module '*.svg' {
  const content: React.FC<React.SVGProps<SVGElement>>
  export default content
}
/// <reference types="vite/client" />
```
:::
## Vue

Vite proporciona soporte Vue de primera clase:

- Compatibilidad con Vue 3 SFC a través de [@vitejs/plugin-vue](https://github.com/vitejs/vite/tree/main/packages/plugin-vue)
- Compatibilidad con Vue 3 JSX a través de [@vitejs/plugin-vue-jsx](https://github.com/vitejs/vite/tree/main/packages/plugin-vue-jsx)
- Compatibilidad con Vue 2.7 a través de [@vitejs/plugin-vue2](https://github.com/vitejs/vite-plugin-vue2)
- Compatibilidad con Vue <2.7 a través de [vite-plugin-vue2](https://github.com/underfin/vite-plugin-vue2)

## JSX

Los archivos `.jsx` y `.tsx` también son compatibles de fábrica. La transpilación JSX también se maneja a través de [esbuild](https://esbuild.github.io).

Los usuarios de Vue deben usar el complemento oficial [@vitejs/plugin-vue-jsx](https://github.com/vitejs/vite/tree/main/packages/plugin-vue-jsx), que proporciona características específicas de Vue 3, incluidas HMR, resolución de componentes globales, directivas y slots.

Si no usas JSX con React o Vue, puedes hacer configuraciones personalizadas de `jsxFactory` y `jsxFragment` usando la [opción `esbuild`](/config/shared-options#esbuild). Por ejemplo para Preact:

```js
// vite.config.js
import { defineConfig } from 'vite'

export default defineConfig({
  esbuild: {
    jsxFactory: 'h',
    jsxFragment: 'Fragment'
  }
})
```

Más detalles en la [documentación de esbuild](https://esbuild.github.io/content-types/#jsx).

Puedes inyectar los helpers de JSX usando `jsxInject` (que es una opción exclusiva de Vite) para evitar las importaciones manuales:

```js
// vite.config.js
import { defineConfig } from 'vite'

export default defineConfig({
  esbuild: {
    jsxInject: `import React from 'react'`
  }
})
```

## CSS

La importación de archivos `.css` inyectará su contenido en la página a través de una etiqueta `<style>` con soporte HMR. También puedes obtener el CSS procesado como una cadena o como exportación predeterminada del módulo.

### Incrustación y rebase de `@import`

Vite está preconfigurado para admitir incrustaciones CSS de `@import` través de `postcss-import`. Los alias de Vite también se respetan para CSS `@import`. Además, todas las referencias de CSS `url()`, incluso si los archivos importados están en directorios diferentes, siempre se reorganizan automáticamente para garantizar la corrección.

Los alias `@import` y el cambio de base de URL también son compatibles con los archivos Sass y Less (consulta los [preprocesadores CSS](#preprocesadores-css)).

### PostCSS

Si el proyecto contiene una configuración de PostCSS válida (cualquier formato compatible con [postcss-load-config](https://github.com/postcss/postcss-load-config), por ejemplo, `postcss.config.js`), se aplicará automáticamente a todo el CSS importado.

### Módulos CSS

Cualquier archivo CSS que termine con `.module.css` se considera un [archivo de módulos CSS](https://github.com/css-modules/css-modules). La importación de dicho archivo devolverá el objeto de módulo correspondiente:

```css
/* example.module.css */
.red {
  color: red;
}
```

```js
import classes from './example.module.css'
document.getElementById('foo').className = classes.red
```

El comportamiento de los módulos CSS se puede configurar mediante la [opción `css.modules`](/config/shared-options#css-modules).

Si `css.modules.localsConvention` está configurado para habilitar camelCase locales (por ejemplo, `localsConvention: 'camelCaseOnly'`), también podrá usar importaciones con nombre:

```js
// .apply-color -> applyColor
import { applyColor } from './example.module.css'
document.getElementById('foo').className = applyColor
```

### Preprocesadores CSS

Debido a que Vite solo se dirige a los navegadores modernos, se recomienda usar variables CSS nativas con complementos de PostCSS que implementen borradores de CSSWG (por ejemplo, [postcss-nesting](https://github.com/csstools/postcss-plugins/tree/main/plugins/postcss-nesting)) y CSS simple y compatible con estándares futuros.

Dicho esto, Vite proporciona soporte integrado para archivos `.scss`, `.sass`, `.less`, `.styl` y `.stylus`. No es necesario instalar complementos específicos de Vite para ellos, pero se debe instalar el preprocesador correspondiente:

```bash
# .scss and .sass
npm add -D sass

# .less
npm add -D less

# .styl and .stylus
npm add -D stylus
```

Si usas componentes de archivo único de Vue (SFC), esto también habilita automáticamente `<style lang="sass">` et al.

Vite mejora la resolución de `@import` para Sass y Less para que también se respeten los alias de Vite. Además, las referencias `url()` relativas dentro de los archivos Sass/Less importados que se encuentran en directorios diferentes del archivo raíz también se reorganizan automáticamente para garantizar la corrección.

El alias `@import` y el cambio de base de URL no son compatibles con Stylus debido a las limitaciones de su API.

También puedes usar módulos CSS combinados con preprocesadores anteponiendo `.module` a la extensión del archivo, por ejemplo `style.module.scss`.

### Deshabilitando la inyección de CSS en la página

La inyección automática de contenido CSS se puede desactivar a través del parámetro de consulta `?inline`. En este caso, la cadena CSS procesada se devuelve como la exportación predeterminada del módulo como de costumbre, pero los estilos no se inyectan en la página.

```js
import styles from './foo.css' // se inyectará en la página
import otherStyles from './bar.css?inline' // no se inyectará en la página
```

## Recursos estáticos

La importación de un recurso estático devolverá la URL pública resuelta cuando se sirva:

```js
import imgUrl from './img.png'
document.getElementById('hero-img').src = imgUrl
```

Las consultas especiales pueden modificar cómo se cargan los recursos:

```js
// carga recursos explícitamente como URL
import assetAsURL from './asset.js?url'
```

```js
// carga recursos como string
import assetAsString from './shader.glsl?raw'
```

```js
// cargar Web Workers
import Worker from './worker.js?worker'
```

```js
// incrustado de Web Workers como cadenas base64 en tiempo de compilado.
import InlineWorker from './worker.js?worker&inline'
```

Más detalles en [Gestión de recursos estáticos](./assets).

## JSON

Los archivos JSON se pueden importar directamente; también se admiten las importaciones con nombre:

```js
// importar todo el objeto
import json from './example.json'
// importe un campo raíz como exportaciones con nombre: ¡ayuda al hacer tree-shaking!
import { field } from './example.json'
```

## Importaciones Glob

Vite admite la importación de múltiples módulos desde el sistema de archivos a través de la función especial `import.meta.glob`:

```js
const modules = import.meta.glob('./dir/*.js')
```

Lo anterior se transformará en lo siguiente:

```js
// código producido por vite
const modules = {
  './dir/foo.js': () => import('./dir/foo.js'),
  './dir/bar.js': () => import('./dir/bar.js')
}
```

A continuación, puedes iterar sobre las claves del objeto `modules` para acceder a los módulos correspondientes:

```js
for (const path in modules) {
  modules[path]().then((mod) => {
    console.log(path, mod)
  })
}
```

Los archivos coincidentes se cargan de forma diferida de forma predeterminada a través de la importación dinámica y se dividirán en partes separadas durante la compilación. Si prefieres importar todos los módulos directamente (por ejemplo, confiando en que los efectos secundarios de estos módulos se apliquen primero), puedes pasar `{ eager: true }` como segundo argumento:

```js
const modules = import.meta.glob('./dir/*.js', { eager: true })
```

Lo anterior se transformará en lo siguiente:

```js
// código producido por vite
import * as __glob__0_0 from './dir/foo.js'
import * as __glob__0_1 from './dir/bar.js'
const modules = {
  './dir/foo.js': __glob__0_0,
  './dir/bar.js': __glob__0_1
}
```

### Formas de importación Glob

`import.meta.glob` también admite la importación de archivos como cadenas (similar a la [importación de recursos como cadena](./assets#importar-recursos-como-cadenas-de-texto)) con la sintaxis de [importación reflexiva](https://github.com/tc39/proposal-import-reflection):

```js
const modules = import.meta.glob('./dir/*.js', { as: 'raw' })
```

Lo anterior se transformará en lo siguiente:

```js
// código producido por vite
const modules = {
  './dir/foo.js': 'export default "foo"\n',
  './dir/bar.js': 'export default "bar"\n'
}
```

`{ as: 'url' }` también se admite para cargar recursos como URL.

### Patrones múltiples

El primer argumento puede ser una array de globs, por ejemplo

```js
const modules = import.meta.glob(['./dir/*.js', './another/*.js'])
```

### Patrones negativos

También se admiten patrones glob negativos (con el prefijo `!`). Para ignorar algunos archivos del resultado, puedes agregar exclusiones de patrones glob en el primer argumento:

```js
const modules = import.meta.glob(['./dir/*.js', '!**/bar.js'])
```

```js
// código producido por vite
const modules = {
  './dir/foo.js': () => import('./dir/foo.js')
}
```

#### Importaciones nombradas

Es posible importar solo partes de los módulos con las opciones de`import`.

```ts
const modules = import.meta.glob('./dir/*.js', { import: 'setup' })
```

```ts
// código producido por vite
const modules = {
  './dir/foo.js': () => import('./dir/foo.js').then((m) => m.setup),
  './dir/bar.js': () => import('./dir/bar.js').then((m) => m.setup)
}
```

Cuando se combina con `eager`, incluso es posible tener habilitado el tree-shaking para esos módulos.

```ts
const modules = import.meta.glob('./dir/*.js', { import: 'setup', eager: true })
```

```ts
// código producido por vite:
import { setup as __glob__0_0 } from './dir/foo.js'
import { setup as __glob__0_1 } from './dir/bar.js'
const modules = {
  './dir/foo.js': __glob__0_0,
  './dir/bar.js': __glob__0_1
}
```

Configura `import` a `default` para importar la exportación predeterminada.

```ts
const modules = import.meta.glob('./dir/*.js', {
  import: 'default',
  eager: true
})
```

```ts
// código producido por vite:
import __glob__0_0 from './dir/foo.js'
import __glob__0_1 from './dir/bar.js'
const modules = {
  './dir/foo.js': __glob__0_0,
  './dir/bar.js': __glob__0_1
}
```

#### Consultas personalizadas

También puedes usar la opción `query` para proporcionar consultas personalizadas a las importaciones para que las consuman otros complementos.

```ts
const modules = import.meta.glob('./dir/*.js', {
  query: { foo: 'bar', bar: true }
})
```

```ts
// código producido por vite:
const modules = {
  './dir/foo.js': () =>
    import('./dir/foo.js?foo=bar&bar=true').then((m) => m.setup),
  './dir/bar.js': () =>
    import('./dir/bar.js?foo=bar&bar=true').then((m) => m.setup)
}
```

### Advertencias de importación glob

Ten en cuenta que:

- Esta es una característica exclusiva de Vite y no es un estándar web o ES.
- Los patrones glob se tratan como especificadores de importación: deben ser relativos (comenzar con `./`) o absolutos (comenzar con `/`, resueltos en relación con la raíz del proyecto) o una ruta de alias (ver [opción `resolve.alias`](/config/shared-options#resolve-alias)).
- La coincidencia de glob se realiza a través de [`fast-glob`](https://github.com/mrmlnc/fast-glob) - Consulta la documentación de los [patrones de glob compatibles](https://github.com/mrmlnc/fast-glob#pattern-syntax).
- También debes tener en cuenta que todos los argumentos en `import.meta.glob` deben **pasarse como literales**. NO puede usar variables o expresiones en ellos.

## Importación dinámica

Similar a [importación de glob](#importaciones-glob), Vite también admite la importación dinámica con variables.

```ts
const module = await import(`./dir/${file}.js`)
```

Ten en cuenta que las variables solo representan nombres de archivo de un nivel de profundidad. Si `file` es `'foo/bar'`, la importación fallaría. Para un uso más avanzado, puede utilizar la función [importación de glob](#importaciones-glob)

## WebAssembly

Los archivos `.wasm` precompilados se pueden importar con `?init`: la exportación predeterminada será una función de inicialización que devuelve una Promesa de la instancia de wasm:

```js
import init from './example.wasm?init'
init().then((instance) => {
  instance.exports.test()
})
```

La función init también puede tomar el objeto `imports` que se pasa a `WebAssembly.instantiate` como su segundo argumento:

```js
init({
  imports: {
    someFunc: () => {
      /* ... */
    }
  }
}).then(() => {
  /* ... */
})
```

En la compilación de producción, los archivos `.wasm` más pequeños que `assetInlineLimit` se insertarán como cadenas base64. De lo contrario, se copiarán en el directorio dist como un recurso y se obtendrán a petición.

::: warning
La [propuesta de integración de módulos ES para WebAssembly](https://github.com/WebAssembly/esm-integration) no es compatible actualmente.
Usa [`vite-plugin-wasm`](https://github.com/Menci/vite-plugin-wasm) u otros complementos de la comunidad para darle el manejo apropiado.
:::

## Web Workers

### Importar con Worker

Se puede importar un script de web worker usando [`new Worker()`](https://developer.mozilla.org/en-US/docs/Web/API/Worker/Worker) y [`new SharedWorker()`](https://developer.mozilla.org/en-US/docs/Web/API/SharedWorker/SharedWorker). En comparación con los sufijos de worker, esta sintaxis se acerca más a los estándares y es la forma **recomendada** de crear workers.

```ts
const worker = new Worker(new URL('./worker.js', import.meta.url))
```

El constructor de worker también acepta opciones, que se pueden usar para crear workers de "módulo":

```ts
const worker = new Worker(new URL('./worker.js', import.meta.url), {
  type: 'module'
})
```

### Importar con sufijos de consulta

Se puede importar directamente un script de web worker agregando `?worker` o `?sharedworker` a la solicitud de importación. La exportación predeterminada será un constructor de worker personalizado:

```js
import MyWorker from './worker?worker'

const worker = new MyWorker()
```

El script del worker también puede usar sentencias `import` en lugar de `importScripts()`; ten en cuenta que durante el desarrollo esto depende del soporte nativo del navegador y actualmente solo funciona en Chrome, pero para la compilación de producción está compilado.

De forma predeterminada, el script del worker se emitirá como un fragmento separado en la compilación de producción. Si deseas listar el worker como cadenas base64, agrega el parámetro `inline`:

```js
import MyWorker from './worker?worker&inline'
```

Si deseas listar el worker como una URL, agrega el parámetro `url`:

```js
import MyWorker from './worker?worker&url'
```

Revisa las [opciones de Worker](/config/worker-options) para obtener detalles sobre cómo configurar el empaquetado de todos los workers.

## Optimizaciones de compilación

> Las funcionalidades que se enumeran a continuación se aplican automáticamente como parte del proceso de compilación y no hay necesidad de una configuración explícita a menos que desees deshabilitarlas.

### División de código CSS

Vite extrae automáticamente el CSS utilizado por los módulos en un fragmento asíncrono y genera un archivo separado para él. El archivo CSS se carga automáticamente a través de una etiqueta `<link>` cuando se carga el fragmento asíncrono asociado, y se garantiza que el fragmento asíncrono solo se evaluará después de cargar el CSS para evitar [FOUC](https://en.wikipedia.org/wiki/Flash_of_unstyled_content#:~:text=A%20flash%20of%20unstyled%20content,before%20all%20information%20is%20retriever.).

Si prefieres que se extraiga todo el CSS en un solo archivo, puedes desactivar la división del código CSS configurando [`build.cssCodeSplit`](/config/build-options#build-csscodesplit) en `false`.

### Generación de directivas de precarga

Vite genera automáticamente directivas `<link rel="modulepreload">` para fragmentos de entrada y sus importaciones directas en el HTML creado.

### Optimización de carga de fragmentos asíncronos

En las aplicaciones del mundo real, Rollup a menudo genera fragmentos "comunes": código que se comparte entre dos o más fragmentos. Combinado con importaciones dinámicas, es bastante común tener el siguiente escenario:

<script setup>
import graphSvg from '../images/graph.svg?raw'
</script>
<svg-image :svg="graphSvg" />

En escenarios no optimizados, cuando se importa el fragmento asíncrono `A`, el navegador tendrá que solicitar y analizar `A` antes de darse cuenta de que también necesita el fragmento común `C`. Esto da como resultado consultas de ida y vuelta adicionales a la red:

```
Entrada ---> A ---> C
```

Vite reescribe automáticamente las llamadas de importación dinámicas de código dividido con un paso de precarga para que cuando se solicite `A`, `C` se obtengan **en paralelo**:

```
Entrada ---> (A + C)
```

Es posible que `C` tenga más importaciones, lo que dará como resultado aún más consultas de ida y vuelta en el escenario no optimizado. La optimización de Vite rastreará todas las importaciones directas para eliminar por completo esas consultas, independientemente de la profundidad de la importación.
