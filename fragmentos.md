# Fragmentos en Android (Java)

Resumen sobre qué son los fragmentos, cómo crearlos y usarlos, y cómo diseñarlos para que sean independientes del proyecto y de la activity que los aloja. Se usa AndroidX (`androidx.fragment.app.Fragment`).

---

## 1. Qué es un fragmento

Un **fragmento** es un componente de interfaz reutilizable que representa una porción de la pantalla, con su propio layout y su propio ciclo de vida. Siempre vive dentro de una activity (o dentro de otro fragmento), y no puede existir por sí solo.

**Para qué sirven**

- Dividir una pantalla en partes reutilizables (lista y detalle, por ejemplo).
- Adaptar la interfaz a distintos tamaños y orientaciones: en portrait un fragmento por pantalla, en landscape o tablet varios a la vez.
- Reutilizar la misma pieza de UI en distintas activities o proyectos.
- Navegar entre pantallas dentro de una sola activity (pila de fragmentos o Navigation Component).

**Características clave**

- Tiene su propio ciclo de vida, que está ligado al de la activity anfitriona.
- Tiene su propio layout y su propia lógica de eventos.
- Se puede añadir, reemplazar o quitar mientras la activity está en marcha.
- Su estado se puede guardar y restaurar por separado.

---

## 2. Ciclo de vida

Orden habitual de callbacks:

| Callback | Para qué se usa |
|---|---|
| `onAttach(Context)` | El fragmento se asocia a la activity. Aquí se obtiene la referencia al listener. |
| `onCreate(Bundle)` | Inicialización que no depende de la vista. Leer `getArguments()`. |
| `onCreateView(...)` | Inflar y devolver el layout. |
| `onViewCreated(View, Bundle)` | La vista ya existe: `findViewById`, adapters, listeners. |
| `onStart()` / `onResume()` | El fragmento es visible / interactivo. |
| `onPause()` / `onStop()` | Deja de ser interactivo / visible. |
| `onDestroyView()` | Se destruye la vista (liberar referencias a ella). |
| `onDestroy()` | Se destruye el fragmento. |
| `onDetach()` | Se desasocia de la activity. Poner el listener a `null`. |

Punto importante: **la vista del fragmento puede destruirse y recrearse mientras el fragmento sigue vivo** (por ejemplo, al estar en la pila de retroceso). Por eso las referencias a vistas deben limpiarse en `onDestroyView()`.

---

## 3. Creación básica

### 3.1 La clase

```java
import android.os.Bundle;
import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;
import android.widget.TextView;

import androidx.annotation.NonNull;
import androidx.annotation.Nullable;
import androidx.fragment.app.Fragment;

public class FrgEjemplo extends Fragment {

    // Constructor vacío obligatorio (el sistema lo usa al recrear el fragmento)
    public FrgEjemplo() { }

    @Nullable
    @Override
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container,
                             @Nullable Bundle savedInstanceState) {
        // attachToRoot SIEMPRE false: el FragmentManager es quien añade la vista
        return inflater.inflate(R.layout.frg_ejemplo, container, false);
    }

    @Override
    public void onViewCreated(@NonNull View view, @Nullable Bundle savedInstanceState) {
        TextView txt = view.findViewById(R.id.txtTitulo);
        txt.setText("Hola desde el fragmento");
    }
}
```

También se puede pasar el layout al constructor del padre y no sobrescribir `onCreateView`:

```java
public FrgEjemplo() {
    super(R.layout.frg_ejemplo);
}
```

### 3.2 El layout (`res/layout/frg_ejemplo.xml`)

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <TextView
        android:id="@+id/txtTitulo"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content" />
</LinearLayout>
```

---

## 4. Uso básico: añadir un fragmento a una activity

### 4.1 Forma estática (en el XML)

El fragmento queda fijo mientras viva el layout. Es la forma que se usa para paneles que siempre están presentes.

```xml
<androidx.fragment.app.FragmentContainerView
    android:id="@+id/frgEjemplo"
    android:name="com.example.app.FrgEjemplo"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```

Se recupera con:

```java
FrgEjemplo f = (FrgEjemplo) getSupportFragmentManager().findFragmentById(R.id.frgEjemplo);
```

### 4.2 Forma dinámica (en código)

Se usa un contenedor vacío en el layout y se coloca el fragmento con una transacción.

```xml
<FrameLayout
    android:id="@+id/contenedor"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```

```java
if (savedInstanceState == null) {   // evita duplicar el fragmento al recrear la activity
    getSupportFragmentManager().beginTransaction()
            .replace(R.id.contenedor, new FrgEjemplo())
            .commit();
}
```

### 4.3 Operaciones de la transacción

| Operación | Efecto |
|---|---|
| `add(id, frg)` | Añade el fragmento al contenedor. |
| `replace(id, frg)` | Sustituye lo que hubiera en el contenedor. |
| `remove(frg)` | Lo quita. |
| `hide(frg)` / `show(frg)` | Lo oculta / lo muestra sin destruirlo. |
| `addToBackStack(nombre)` | Permite volver al estado anterior con el botón Atrás. |
| `commit()` | Ejecuta la transacción de forma asíncrona. |

Ejemplo con pila de retroceso:

```java
getSupportFragmentManager().beginTransaction()
        .replace(R.id.contenedor, new FrgOtro())
        .addToBackStack(null)
        .commit();
```

### 4.4 Pasar datos al fragmento: `Bundle` de argumentos

No se deben pasar datos con constructores con parámetros ni con setters, porque se perderían al recrear el fragmento. Se usa un `Bundle` y, por convención, un método estático `newInstance`.

```java
public class FrgDetalle extends Fragment {

    private static final String ARG_ID = "id";

    public static FrgDetalle newInstance(int id) {
        FrgDetalle f = new FrgDetalle();
        Bundle args = new Bundle();
        args.putInt(ARG_ID, id);
        f.setArguments(args);       // el sistema conserva los arguments al recrear
        return f;
    }

    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        int id = requireArguments().getInt(ARG_ID);
        // ...
    }
}
```

### 4.5 Guardar estado

Para el estado que cambia mientras se usa el fragmento:

```java
@Override
public void onSaveInstanceState(@NonNull Bundle outState) {
    super.onSaveInstanceState(outState);
    outState.putInt("idActual", idActual);
}
```

Se restaura leyendo `savedInstanceState` en `onCreate` o en `onViewCreated`.

---

## 5. Diseño de fragmentos independientes del proyecto

El objetivo es que un fragmento se pueda copiar a otro proyecto, o usar en otra activity, **sin modificarlo**. Para ello debe cumplir estas reglas.

### Regla 1: no conocer a la activity concreta

Mal:


~~((MainActivity) getActivity()).cargarDetalle(id);   // acoplado a MainActivity~~


Bien: definir una **interfaz de callback** dentro del propio fragmento y que la activity la implemente.

```java
public class FrgLista extends Fragment {

    public interface OnItemSeleccionadoListener {
        void onItemSeleccionado(int id);
    }

    private OnItemSeleccionadoListener listener;

    @Override
    public void onAttach(@NonNull Context context) {
        super.onAttach(context);
        if (context instanceof OnItemSeleccionadoListener) {
            listener = (OnItemSeleccionadoListener) context;
        } else {
            throw new IllegalStateException(
                context.getClass().getSimpleName() + " debe implementar OnItemSeleccionadoListener");
        }
    }

    @Override
    public void onDetach() {
        super.onDetach();
        listener = null;     // evita fugas de memoria
    }
}
```

La activity implementa la interfaz y decide qué hacer. El fragmento solo avisa de que ha pasado algo.

### Regla 2: no depender de otros fragmentos

Un fragmento no debe buscar ni llamar directamente a otro fragmento. Toda la comunicación pasa por la activity (o por un `ViewModel` compartido, o por la Fragment Result API). Así, la activity es la única que conoce la "composición" de la pantalla.

### Regla 3: recibir los datos por `arguments`, nunca por el Intent

`getActivity().getIntent()` ata el fragmento a una activity concreta. Los datos de entrada se reciben con `setArguments()` / `newInstance()`, y quien aloje el fragmento los obtiene de donde quiera (Intent, base de datos, etc.).

### Regla 4: no asumir el entorno de pantalla

El fragmento no debe saber si hay otro panel al lado, si está en portrait o en landscape, ni si es tablet. Esa decisión la toma la activity mediante los layouts (`layout/`, `layout-land/`, `layout-sw600dp/`...).

### Regla 5: layout y recursos propios

Cada fragmento trae su layout (`frg_xxx.xml`), sus ids y sus strings. No debe usar `findViewById` de ids que pertenezcan al layout de la activity. Hay que buscar siempre desde la vista del fragmento: `view.findViewById(...)`.

### Regla 6: constructor vacío y `newInstance`

Mantener siempre el constructor sin parámetros, y ofrecer `newInstance(...)` como API pública para crearlo con sus datos.

### Regla 7: API pública mínima y clara

Exponer solo lo necesario: el `newInstance`, la interfaz de callback y, si hace falta, métodos para actualizarlo desde fuera (por ejemplo, `mostrar(int id)`). Todo lo demás, privado.

### Regla 8: no guardar referencias que sobrevivan a la vista

- Poner el listener a `null` en `onDetach()`.
- Si se usa ViewBinding, poner el `binding` a `null` en `onDestroyView()`.
- No guardar `Context` de activity en campos estáticos.

### Regla 9: usar el contexto adecuado

`requireContext()` y `requireActivity()` en lugar de `getContext()` / `getActivity()` cuando se da por hecho que están asociados, para fallar con un mensaje claro y no con un `NullPointerException` confuso.

---

## 6. Formas de comunicación entre fragmentos y activity

| Mecanismo | Cuándo usarlo |
|---|---|
| **Interfaz de callback** (fragmento → activity) | Caso simple y muy explícito. Es el que se usa en el ejemplo lista/detalle. |
| **Llamada de la activity a un método público del fragmento** (activity → fragmento) | Cuando la activity debe actualizar un fragmento ya visible, por ejemplo `detalle.mostrar(id)`. |
| **Fragment Result API** | Comunicación entre fragmentos, o fragmento → activity, sin interfaces ni referencias directas. |
| **ViewModel compartido** (`ViewModelProvider(requireActivity())`) | Cuando varios fragmentos comparten datos u observan el mismo estado. |

### La interfaz de callback en detalle: qué viaja en cada dirección

Una interfaz de callback es el **contrato** que el fragmento define para hablar con el exterior sin saber quién hay al otro lado. Cada método de la interfaz es un canal de comunicación, y en él viajan datos en las dos direcciones:

| Parte del método | Dirección del dato | Significado |
|---|---|---|
| **Parámetros** | Del fragmento **hacia fuera** | Información que el fragmento entrega al exterior: qué ha pasado y con qué datos (el id seleccionado, el texto escrito, etc.). |
| **Tipo devuelto** | De fuera **hacia dentro** del fragmento | Información que el exterior devuelve al fragmento para que **tome una decisión** (puedo continuar o no, qué modo uso, qué valor muestro...). |

Según el tipo devuelto, el método cumple un papel distinto:

- **`void`**: solo notificación. El fragmento avisa de que algo ha ocurrido y no necesita respuesta.
- **`boolean`**: autorización o veto. El exterior decide si el fragmento puede seguir adelante con una acción.
- **Otro tipo (`int`, `String`, un objeto...)**: consulta. El fragmento pide un dato al exterior para usarlo en su lógica.

Ejemplo de interfaz con los tres casos:

```java
public class FrgLista extends Fragment {

    public interface OnListaListener {

        // void: solo notificación.
        // El parámetro viaja del fragmento hacia fuera.
        void onItemSeleccionado(int id);

        // boolean: el exterior autoriza o veta.
        // El parámetro (id) sale del fragmento; el retorno entra al fragmento
        // y decide si se borra el elemento.
        boolean onSolicitarBorrado(int id);

        // Tipo devuelto distinto de void: consulta.
        // No hay parámetros; el exterior responde con un dato
        // que el fragmento usa para filtrar su lista.
        String onObtenerFiltro();
    }

    private OnListaListener listener;

    // ... onAttach / onDetach como en el punto 5 ...

    private void intentarBorrar(Item item) {
        // El fragmento pregunta al exterior y actúa según la respuesta
        boolean permitido = (listener != null) && listener.onSolicitarBorrado(item.id);
        if (permitido) {
            adapter.remove(item);
        }
    }

    private void aplicarFiltro() {
        String filtro = (listener != null) ? listener.onObtenerFiltro() : "";
        adapter.getFilter().filter(filtro);
    }
}
```

La activity que aloja el fragmento implementa esos métodos y decide qué responder:

```java
public class MainActivity extends AppCompatActivity implements FrgLista.OnListaListener {

    @Override
    public void onItemSeleccionado(int id) { /* mostrar detalle */ }

    @Override
    public boolean onSolicitarBorrado(int id) {
        return usuarioEsAdministrador;      // la decisión es de la activity
    }

    @Override
    public String onObtenerFiltro() {
        return textoDeBusqueda;             // el dato viene de fuera
    }
}
```

Puntos a tener en cuenta:

- **El fragmento hace la pregunta y el exterior pone la respuesta.** El fragmento nunca sabe cómo se decide (rol del usuario, ajustes, estado de otra pantalla...), solo usa el resultado. Eso es lo que mantiene su independencia.
- **Prever siempre un valor por defecto cuando no hay listener** (`listener == null`), para que el fragmento tenga un comportamiento razonable por sí mismo.
- **Los métodos con retorno son síncronos**: la respuesta debe ser inmediata. Si la decisión requiere algo lento o asíncrono (diálogo, red), conviene usar `void` y que la respuesta vuelva más tarde por otro método del fragmento, por la Fragment Result API o por un `ViewModel`.
- **Mantener los parámetros y retornos simples y genéricos** (ids, textos, booleanos, tipos propios del fragmento). Si se pasan tipos de la activity o del proyecto, el fragmento deja de ser reutilizable.

### Fragment Result API (ejemplo)

Fragmento que envía el resultado:

```java
Bundle result = new Bundle();
result.putInt("id", idSeleccionado);
getParentFragmentManager().setFragmentResult("seleccion", result);
```

Activity (o fragmento) que lo recibe:

```java
getSupportFragmentManager().setFragmentResultListener(
        "seleccion", this, (requestKey, bundle) -> {
            int id = bundle.getInt("id");
            // reaccionar
        });
```

### ViewModel compartido (ejemplo)

```java
public class SeleccionViewModel extends ViewModel {
    private final MutableLiveData<Integer> idSeleccionado = new MutableLiveData<>();
    public void seleccionar(int id) { idSeleccionado.setValue(id); }
    public LiveData<Integer> getIdSeleccionado() { return idSeleccionado; }
}
```

```java
// En cualquier fragmento (mismo scope: la activity)
SeleccionViewModel vm = new ViewModelProvider(requireActivity()).get(SeleccionViewModel.class);
vm.getIdSeleccionado().observe(getViewLifecycleOwner(), id -> { /* actualizar UI */ });
```

---

## 7. Patrón lista/detalle adaptable (resumen)

- `FrgLista` avisa mediante `OnItemSeleccionadoListener`.
- `FrgDetalle` recibe el id por `arguments` o con `mostrar(id)`.
- La `MainActivity` implementa el listener y comprueba si el fragmento de detalle está en pantalla:
  - **Si está** (layout landscape con los dos paneles): llama a `detalle.mostrar(id)`.
  - **Si no está** (layout portrait con solo la lista): lanza `DetalleActivity` con el id en el Intent, y esta crea el fragmento con `FrgDetalle.newInstance(id)`.
- Los fragmentos no cambian: lo único que cambia es qué layout se carga (`layout/` o `layout-land/`).

---

## 8. Errores frecuentes

- **Pasar datos por constructor con parámetros**: se pierden al recrear el fragmento. Usar `arguments`.
- **Inflar con `attachToRoot = true`** dentro de `onCreateView`: provoca una excepción, porque el `FragmentManager` ya añade la vista.
- **Duplicar el fragmento al rotar**: olvidar el `if (savedInstanceState == null)` antes de añadirlo por código.
- **Usar `getActivity()` cuando ya no hay activity** (por ejemplo, en un callback asíncrono tardío): puede ser `null`.
- **Acceder a las vistas en `onCreate`**: todavía no existen. Hacerlo en `onViewCreated`.
- **Referencias a vistas tras `onDestroyView`**: fugas de memoria y errores. Limpiarlas.
- **Acoplar el fragmento a una activity concreta** con casts (`(MainActivity) getActivity()`): rompe la reutilización.
- **Olvidar `addToBackStack`** cuando se quiere poder volver con el botón Atrás.

---

## 9. Checklist de un fragmento independiente

- [ ] Constructor público vacío y método `newInstance(...)` con `Bundle` de argumentos.
- [ ] Layout propio y ids propios; `findViewById` siempre desde la vista del fragmento.
- [ ] Interfaz de callback interna para comunicarse hacia fuera; comprobada en `onAttach` y puesta a `null` en `onDetach`.
- [ ] No hay casts a una activity concreta ni referencias a otros fragmentos.
- [ ] No lee el `Intent` de la activity.
- [ ] No asume orientación, tamaño de pantalla ni presencia de otros paneles.
- [ ] Guarda su estado en `onSaveInstanceState` si lo necesita.
- [ ] Libera referencias a la vista en `onDestroyView`.
- [ ] API pública mínima y documentada.
