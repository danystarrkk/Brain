----

## Inferencias lógicas

Las leyes de equivalencia y los medios de comunicación.

Existen diferentes mecanismos de inferencia lógica, siendo el fundamental el método deductivo, basado en la ley modus ponens. También existe un mecanismo poderoso conocido como principio de resolución, mismo que utiliza razonamientos
basados en refutación.

Es evidente que para realizar un proceso de inferencia lógica valido, las aseveraciones deben referirse a un mismo contexto y debe haber una relación intima entre ellas .
Para utilizar el método deductivo nos podemos apoyar en los denominando encadenamientos que son de dos tipos:

- Hacia adelante (feet forward): este permite ir sustituyendo los antecedentes de las reglas para la generación de las conclusiones. 
- Hacia atrás (back forward): este parte de las conclusiones tratando de cubrir los antecedentes.

> [!Ejemplos] Considere las siguientes aseveraciones y realize las demostraciones solicitadas.

1. A john le gusta toda clase de comida.
	- (∀x)(comida(x) -> gusta("john",x))

2. Las manzanas son comida.
	- comida("manzanas")

3. El pollo es comida.
	- comida("pollo")

4. cualquier cosa que uno coma y no le mate es comida.
	- (∀x)(∀y)(come(x,y) ^ vivo(x) -> comida(y))

5. Bill come cacahuetes y aún está vivo.
	- come("Bill", "cacahuete"), vivo("Bill")

6.  Sue come todo lo que come Bill.
	- (∀x)(come("Bill", x) -> come("Sue", x))

Demostrar que a John le gusta los cacahuetes. -> *PD: gusta("john", "cacahuetes")*


## El principio de resolución

Este método permite realizar demostraciones mediante refutación (Hay que negar lo que es pretende demostrar). El único requerimiento es que las codificaciones deben ser traducidas a la forma **Clausal**, es decir a **Conjunción de Disyunciones**. Ejemplos:

- `(P1 ∨ P2) ^ (P2 ∨ P1 ∨ P2) ∨......`
- `P1 ^ P2 ^ P3`

una vez realizada las transformaciones de la forma clausal hay que comenzar a determinar mediante combinaciones clausiales los denominados resolventes, debiendo utilizar en algún momento la negación de lo que se debe de utilizar. El proceso termina cuando se encuentra el resolvente vacío.

> [!Ejemplo] Considere las siguientes aseveraciones y demuestre lo solicitado

**p** -> un triangulo tiene 3 ángulos.
**q** -> un cuadro tiene 4 ángulos rectos.
**r** -> la suma vale dos ángulos rectos.
**s** -> los rombos tienen cuatro ángulos rectos.

Demostrar que los rombos no tiene cuatro ángulos rectos.

![[Pasted image 20260416121732.png]]
Demostración:
![[Pasted image 20260416121813.png]]

Se demostró que los rombos si tiene 4 ángulos recto.


demostrar que 