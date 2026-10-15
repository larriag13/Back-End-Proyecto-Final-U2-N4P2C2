# Back-End-Proyecto-Final-U2-N4P2C2
### Context Processor para el Carrito
Crea el archivo `apps/pedidos/context_processors.py`:
```python
def carrito_context(request):
    """Permite acceder al número de figuras en el carrito desde cualquier plantilla."""
    carrito = request.session.get('carrito', {})
    total_items = sum(item.get('cantidad', 0) for item in carrito.values())
    return {
        'carrito_total_items': total_items
    }
```

---

## Capa de Datos — Django ORM y Modelos Relacionales

### Fundamento Técnico: Django ORM y Claves Foráneas
El ORM traduce clases de Python en tablas de base de datos relacionales. En este modelo:
* **`Categoria`:** Franquicia o universo temático.
* **`Figura`:** Entidad principal que almacena los atributos del coleccionable. La relación `ForeignKey` vincula muchas figuras a una categoría.
* **`on_delete=models.PROTECT`:** Regla de integridad referencial que impide borrar una categoría si existen figuras vinculadas a ella, evitando registros huérfanos.

Editar `apps/catalogo/models.py`:

```python
from django.db import models
from django.utils.text import slugify
from django.core.validators import MinValueValidator


class Categoria(models.Model):
    nombre = models.CharField(max_length=100, unique=True, verbose_name="Categoría o Franquicia")
    slug = models.SlugField(max_length=120, unique=True, blank=True, help_text="URL amigable")
    descripcion = models.TextField(blank=True, verbose_name="Descripción")
    activo = models.BooleanField(default=True, verbose_name="¿Activo?")
    creado_en = models.DateTimeField(auto_now_add=True)

    class Meta:
        verbose_name_plural = "Categorías"
        ordering = ['nombre']

    def __str__(self):
        return self.nombre

    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.nombre)
        super().save(*args, **kwargs)


class Figura(models.Model):
    nombre = models.CharField(max_length=200, verbose_name="Nombre de la Figura")
    slug = models.SlugField(max_length=220, unique=True, blank=True)
    categoria = models.ForeignKey(Categoria, on_delete=models.PROTECT, related_name="figuras", verbose_name="Categoría")
    fabricante = models.CharField(max_length=100, verbose_name="Fabricante / Marca")
    escala = models.CharField(max_length=50, blank=True, verbose_name="Escala o Altura")
    descripcion = models.TextField(verbose_name="Descripción y Materiales")
    precio = models.DecimalField(max_digits=10, decimal_places=0, validators=[MinValueValidator(0)], verbose_name="Precio en CLP")
    stock = models.PositiveIntegerField(default=0, verbose_name="Stock disponible")
    imagen = models.ImageField(upload_to="figuras/", blank=True, null=True, verbose_name="Fotografía")
    destacado = models.BooleanField(default=False, verbose_name="¿Destacado?")
    activo = models.BooleanField(default=True, verbose_name="¿Disponible?")
    fecha_creacion = models.DateTimeField(auto_now_add=True)
    fecha_actualizacion = models.DateTimeField(auto_now=True)

    class Meta:
        verbose_name_plural = "Figuras Coleccionables"
        ordering = ['-fecha_creacion']

    def __str__(self):
        return f"{self.nombre} ({self.fabricante})"

    @property
    def en_stock(self):
        return self.stock > 0

    @property
    def precio_formateado(self):
        return f"${self.precio:,.0f}".replace(",", ".")

    def save(self, *args, **kwargs):
        if not self.slug:
            base_slug = slugify(self.nombre)
            slug = base_slug
            contador = 1
            while Figura.objects.filter(slug=slug).exclude(pk=self.pk).exists():
                slug = f"{base_slug}-{contador}"
                contador += 1
            self.slug = slug
        super().save(*args, **kwargs)
```

### Ejecutar Migraciones
Crea los archivos de migración y aplícalos a la base de datos:
```bash
python manage.py makemigrations
python manage.py migrate
```