
## Clase final alternativa - COMPARAR CON ANTERIOR

```java
import android.os.Bundle;
import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;
import android.widget.Button;

import androidx.annotation.NonNull;
import androidx.annotation.Nullable;
import androidx.fragment.app.Fragment;

public class FrgBotonLimitado extends Fragment {

    private static final String STATE_NUM_CLICS = "numClics";

    private Button boton;
    private int numClics, maxClics;
    private String texto;
    private OnClicLimitadoListener listener;

    // region Interface OnClicLimitadoListener
    /** Escucha los clics de un botón con número máximo de pulsaciones. */
    public interface OnClicLimitadoListener {

        /**
         * @param numClic  número de este clic (1 para el primero)
         * @param maxClics máximo de clics permitidos
         * @return true si el clic se acepta y debe contarse;
         *         false para rechazarlo (no consume uno de los clics disponibles)
         */
        boolean onClick(int numClic, int maxClics);

        /** Se llama justo después del último clic permitido, con el botón ya deshabilitado. */
        void onUltimoClic();
    }
    // endregion

    // region API pública
    /**
     * Configura el botón. Debe llamarse en cada creación de la actividad (también tras
     * un giro de pantalla), porque el listener, el máximo y el texto no se guardan.
     * No reinicia el contador: para eso está {@link #reiniciar()}.
     */
    public void setOnClickListener(OnClicLimitadoListener listener, int maxClics, String texto) {
        this.listener = listener;
        this.maxClics = maxClics;
        setText(texto);
        actualizarBoton();
    }

    public void setText(String texto) {
        this.texto = texto;
        if (boton != null) boton.setText(texto);
    }

    /** Vuelve a dejar disponibles todos los clics y rehabilita el botón. */
    public void reiniciar() {
        numClics = 0;
        actualizarBoton();
    }

    /** Clics aceptados hasta ahora. */
    public int getNumClics() { return numClics; }

    /** Máximo de clics configurado (0 si aún no se ha configurado). */
    public int getMaxClics() { return maxClics; }
    // endregion

    // region Ciclo de vida
    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        if (savedInstanceState != null) {
            numClics = savedInstanceState.getInt(STATE_NUM_CLICS);
        }
    }

    @Nullable
    @Override
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container,
                             @Nullable Bundle savedInstanceState) {
        View layout = inflater.inflate(R.layout.frg_boton, container, false);
        boton = layout.findViewById(R.id.botonFrg);
        boton.setOnClickListener(v -> botonFrgClic());
        if (texto != null) boton.setText(texto);
        actualizarBoton();
        return layout;
    }

    @Override
    public void onSaveInstanceState(@NonNull Bundle outState) {
        super.onSaveInstanceState(outState);
        outState.putInt(STATE_NUM_CLICS, numClics);
    }

    @Override
    public void onDestroyView() {
        super.onDestroyView();
        boton = null;
    }

    @Override
    public void onDetach() {
        super.onDetach();
        listener = null;
    }
    // endregion

    // region Lógica interna
    private void botonFrgClic() {
        if(listener!=null)
            if(numClics< maxClics) {
                if(listener.onClick(numClics+1, maxClics))
                    if((++numClics== maxClics)) {
                        actualizarBoton();
                        listener.onUltimoClic();
                    }
            }
    }

    /** El botón solo está habilitado si hay listener y quedan clics disponibles. */
    private void actualizarBoton() {
        if (boton != null) boton.setEnabled(listener != null && numClics < maxClics);
    }
    // endregion
}
```

## Cómo usarlo desde una actividad

**Layout** (fragmento estático):

```xml
<androidx.fragment.app.FragmentContainerView
    android:id="@+id/frgBoton"
    android:name="com.example.app.FrgBotonLimitado"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content" />
```

**Actividad:**

```java
public class MainActivity extends AppCompatActivity
        implements FrgBotonLimitado.OnClicLimitadoListener {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        FrgBotonLimitado frg = (FrgBotonLimitado) getSupportFragmentManager()
                .findFragmentById(R.id.frgBoton);

        // Siempre, también tras girar la pantalla (el contador sí se conserva)
        frg.setOnClickListener(this, 3, "Pulsar");
    }

    @Override
    public boolean onClick(int numClic, int maxClics) {
        Toast.makeText(this, "Clic " + numClic + " de " + maxClics, Toast.LENGTH_SHORT).show();
        return true;   // true: el clic cuenta; false: se rechaza y no consume un clic
    }

    @Override
    public void onUltimoClic() {
        Toast.makeText(this, "Era el último clic", Toast.LENGTH_SHORT).show();
    }
}
```
