# Laboratorio 02: Sumador de 1 bit, sumador de 4 bits y sumador/restador

Diseño de sistemas digitales combinacionales en Verilog (Quartus) e implementación en FPGA.

## Contenido

1. [Objetivos](#1-objetivos)
2. [Parte 1: Sumador de 1 bit y de 4 bits](#2-parte-1-sumador-de-1-bit-y-de-4-bits)
3. [Parte 2: Sumador/restador de 4 bits](#3-parte-2-sumadorrestador-de-4-bits)
4. [Reto: Control térmico de caldera](#4-reto-control-térmico-de-caldera-industrial)
5. [Simulación y pruebas](#5-simulación-y-pruebas)
6. [Implementación en la FPGA](#6-implementación-en-la-fpga)
7. [Estructura del repositorio](#7-estructura-del-repositorio)

---

## 1. Objetivos

- Diseñar y construir sistemas digitales combinacionales, enfocados en sumadores.
- Describir, instanciar y reutilizar módulos con un lenguaje de descripción de hardware (HDL).
- Recorrer el flujo completo de diseño: especificación, simulación, síntesis e implementación en FPGA.
- Implementar una resta mediante complemento a 2 reutilizando un sumador de 4 bits.
- Verificar el diseño en simulación antes de llevarlo al hardware.

---

## 2. Parte 1: Sumador de 1 bit y de 4 bits

### 2.1 Sumador completo de 1 bit

Circuito combinacional que suma dos bits (`A`, `B`) y un acarreo de entrada (`Ci`), y entrega el bit de suma (`So`) y el acarreo de salida (`Co`).

| A | B | Ci | Co | So |
|---|---|----|----|----|
| 0 | 0 | 0  | 0  | 0  |
| 0 | 0 | 1  | 0  | 1  |
| 0 | 1 | 0  | 0  | 1  |
| 0 | 1 | 1  | 1  | 0  |
| 1 | 0 | 0  | 0  | 1  |
| 1 | 0 | 1  | 1  | 0  |
| 1 | 1 | 0  | 1  | 0  |
| 1 | 1 | 1  | 1  | 1  |

Con mapas de Karnaugh se obtienen las ecuaciones:

```
So = A ⊕ B ⊕ Ci
Co = A·B + Ci·(A ⊕ B)
```

**Implementación en HDL.** Se puede describir de forma estructural de dos maneras:

- **Operadores lógicos** (`&`, `|`, `~`, `^`): más abstractos y concisos; el sintetizador decide cómo mapearlos a compuertas.
- **Primitivas** (`and`, `or`, `xor`): más cercanas al hardware, cada una corresponde a una compuerta concreta.

En este laboratorio se usan operadores lógicos con `assign`.

```verilog
module sumador_1bit(
    input  A,
    input  B,
    input  Ci,
    output S,
    output Co
);
    // Suma
    assign S  = A ^ B ^ Ci;
    // Acarreo de salida
    assign Co = (A & B) | (Ci & (A ^ B));
endmodule
```

### 2.2 Sumador de 4 bits

Se construye **instanciando cuatro sumadores de 1 bit** en cascada (diseño estructural y modular). El acarreo de salida de cada etapa es el acarreo de entrada de la siguiente (*ripple carry*); los bits de `A` y `B` se procesan en paralelo.

Sintaxis de instanciación en Verilog:

```verilog
nombre_modulo nombre_instancia (.puerto_0(señal_0), ..., .puerto_n(señal_n));
```

```verilog
module sumador_4bits(
    input  [3:0] A,
    input  [3:0] B,
    input        Ci,
    output [3:0] S,
    output       Co
);
    // Cables internos para propagar el acarreo
    wire c1, c2, c3;

    sumador_1bit bit0 (.A(A[0]), .B(B[0]), .Ci(Ci), .S(S[0]), .Co(c1));
    sumador_1bit bit1 (.A(A[1]), .B(B[1]), .Ci(c1), .S(S[1]), .Co(c2));
    sumador_1bit bit2 (.A(A[2]), .B(B[2]), .Ci(c2), .S(S[2]), .Co(c3));
    sumador_1bit bit3 (.A(A[3]), .B(B[3]), .Ci(c3), .S(S[3]), .Co(Co));
endmodule
```

---

## 3. Parte 2: Sumador/restador de 4 bits

### 3.1 Complemento a 2

La resta se convierte en suma:

```
A − B = A + (~B + 1)
```

1. **Paso 1:** invertir todos los bits de `B` (complemento a 1).
2. **Paso 2:** sumar 1 al resultado.

**Ejemplo, 7 − 5:**

```
 5 = 0101  →  ~5 = 1010  →  +1 = 1011  (−5)
 0111 (7) + 1011 (−5) = 1 0010  →  descartando el acarreo: 0010 = 2
```

### 3.2 Circuito

- Compuertas **XOR** sobre `B`: con `Sel = 1` invierten los bits de `B`; con `Sel = 0` lo dejan pasar.
- `Sel` también se conecta al acarreo inicial `Ci` del primer sumador, aportando el «+1» del complemento a 2.
- Resultado: `Sel = 0` → `A + B`; `Sel = 1` → `A − B`.
- Interpretación del acarreo final `Co` en una resta: `Co = 1` → resultado positivo o cero; `Co = 0` → resultado negativo (en complemento a 2). Ejemplo: `3 − 7 = 1100` → −4.

```verilog
module sumador_restador_4bits(
    input  [3:0] A,
    input  [3:0] B,
    input        Sel,   // 0 = suma, 1 = resta
    output [3:0] S,
    output       Co
);
    wire [3:0] B_xor;
    wire c1, c2, c3;

    // XOR condicional sobre B
    assign B_xor = B ^ {4{Sel}};

    // Sel actúa como acarreo inicial
    sumador_1bit bit0 (.A(A[0]), .B(B_xor[0]), .Ci(Sel), .S(S[0]), .Co(c1));
    sumador_1bit bit1 (.A(A[1]), .B(B_xor[1]), .Ci(c1),  .S(S[1]), .Co(c2));
    sumador_1bit bit2 (.A(A[2]), .B(B_xor[2]), .Ci(c2),  .S(S[2]), .Co(c3));
    sumador_1bit bit3 (.A(A[3]), .B(B_xor[3]), .Ci(c3),  .S(S[3]), .Co(Co));
endmodule
```

> **Nota sobre el código original:** en el archivo del laboratorio faltaba el `endmodule` de `sumador_1bit` y sobraba un `endmodule` al final del archivo; ambos errores impiden compilar en Quartus. Aquí ya están corregidos. Además, el puerto de control se llama `Sel` (como en la guía) en lugar de `R`, para no confundirlo con el resultado `R = T − L` del reto.

---

## 4. Reto: Control térmico de caldera industrial

Una caldera debe calentar una mezcla hasta una temperatura objetivo. Se requiere un sistema hardware, independiente del software principal, que indique el estado del proceso.

### Entradas

| Señal   | Descripción                              |
|---------|------------------------------------------|
| `T[3:0]`| Temperatura medida en la caldera         |
| `L[3:0]`| Temperatura objetivo del proceso         |
| `Sel`   | Control del sumador/restador (modo resta)|

### Salidas

| Señal         | Significado                              |
|---------------|------------------------------------------|
| `PERFECT_LED` | Temperatura perfecta                     |
| `PROCESS_LED` | Producto aún en proceso de fabricación   |
| `BURN_LED`    | Producto quemado por exceso de temperatura|

### Comportamiento (R = T − L)

| Resultado | Estado                      | LED activo    |
|-----------|-----------------------------|---------------|
| R < 0     | Producto en fabricación     | `PROCESS_LED` |
| R = 0     | Condición ideal             | `PERFECT_LED` |
| R > 0     | Producto quemado            | `BURN_LED`    |

### Restricciones

- Usar el sumador/restador del laboratorio.
- No usar comparadores dedicados.
- No usar microcontroladores.
- Solo lógica combinacional adicional.

### Estrategia de diseño

Con `Sel = 1` el sumador/restador calcula `S = T − L` (4 bits) y entrega `Co`:

- `Co = 0` → hubo «préstamo» → `T < L` → **R < 0**.
- `Co = 1` y `S == 0000` → **R = 0**.
- `Co = 1` y `S != 0000` → **R > 0**.

Las salidas se obtienen con un poco de lógica adicional (NOR de los bits de `S` y compuertas AND/NOT):

```
zero         = ~(S[3] | S[2] | S[1] | S[0])
PROCESS_LED  = ~Co
PERFECT_LED  =  Co &  zero
BURN_LED     =  Co & ~zero
```

### Propuesta de módulo

```verilog
module control_caldera(
    input  [3:0] T,
    input  [3:0] L,
    input        Sel,          // debe ser 1 para que el circuito reste
    output       PERFECT_LED,
    output       PROCESS_LED,
    output       BURN_LED
);
    wire [3:0] R;
    wire       Co;
    wire       zero;

    // R = T - L (reutiliza el sumador/restador)
    sumador_restador_4bits resta (
        .A(T), .B(L), .Sel(Sel), .S(R), .Co(Co)
    );

    assign zero = ~(R[3] | R[2] | R[1] | R[0]);

    assign PROCESS_LED = ~Co;
    assign PERFECT_LED =  Co &  zero;
    assign BURN_LED    =  Co & ~zero;
endmodule
```

> Si `Sel = 0` el circuito suma en lugar de restar y los LEDs dejan de tener sentido. Se recomienda dejar `Sel` en `1` (por ejemplo, un switch fijo o una constante `1'b1` en la instancia superior).

---

## 5. Simulación y pruebas

Casos mínimos a verificar en el testbench:

| Caso                     | T    | L    | R = T − L | Co | Salida esperada |
|--------------------------|------|------|-----------|----|-----------------|
| Temperatura menor        | 0011 | 0111 | 1100 (−4) | 0  | `PROCESS_LED`   |
| Temperatura igual        | 0101 | 0101 | 0000      | 1  | `PERFECT_LED`   |
| Temperatura mayor        | 0111 | 0101 | 0010 (+2) | 1  | `BURN_LED`      |

Casos adicionales sugeridos para el sumador/restador: `7 − 5 = 2`, `3 − 7 = −4`, `A + B` con acarreo final y operandos en 0 o 15.

Ejemplo de testbench para el reto:

```verilog
`timescale 1ns/1ps
module tb_control_caldera;
    reg  [3:0] T, L;
    reg        Sel;
    wire       PERFECT_LED, PROCESS_LED, BURN_LED;

    control_caldera dut (
        .T(T), .L(L), .Sel(Sel),
        .PERFECT_LED(PERFECT_LED),
        .PROCESS_LED(PROCESS_LED),
        .BURN_LED(BURN_LED)
    );

    initial begin
        $monitor("T=%b L=%b | PROCESS=%b PERFECT=%b BURN=%b",
                 T, L, PROCESS_LED, PERFECT_LED, BURN_LED);
        Sel = 1;
        T = 4'b0011; L = 4'b0111; #10;  // menor
        T = 4'b0101; L = 4'b0101; #10;  // igual
        T = 4'b0111; L = 4'b0101; #10;  // mayor
        $finish;
    end
endmodule
```

---

## 6. Implementación en la FPGA

1. Crear un proyecto en **Quartus** y agregar los archivos `.v`.
2. Definir como *top-level* el módulo a probar (`sumador_4bits`, `sumador_restador_4bits` o `control_caldera`).
3. Asignar pines (Pin Planner) según la tarjeta de desarrollo: switches para `A`/`T`, `B`/`L` y `Sel`; LEDs para `S`, `Co` y las salidas del reto.
4. Compilar (síntesis, *fitter* y *assembler*).
5. Programar la FPGA con el archivo `.sof` desde el *Programmer*.
6. Verificar el funcionamiento en la tarjeta con los casos de prueba de la sección anterior.

---

## 7. Estructura del repositorio

```
.
├── README.md
├── sumador_1bit.v
├── sumador_4bits.v
├── sumador_restador_4bits.v
├── control_caldera.v
└── tb_control_caldera.v
```

## Entregables

1. Descripción de hardware de las partes 1 y 2 (y del reto).
2. Explicación del diseño (este documento).
3. Implementación en la tarjeta de desarrollo con Quartus y demostración en laboratorio.
