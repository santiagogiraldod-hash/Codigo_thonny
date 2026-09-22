from machine import Pin, ADC, PWM
import time
import random
import sys
import select

# ==========================================
# ESTADOS DE LA FSM
# ==========================================
ESTADO_IDLE = 0
ESTADO_READY = 1
ESTADO_NEW_CHALLENGE = 2
ESTADO_WAITING_REACTION = 3
ESTADO_HIT = 4
ESTADO_MISS = 5
ESTADO_MEM_INIT = 6
ESTADO_MEM_SHOW_SEQUENCE = 7
ESTADO_MEM_WAITING_INPUT = 8
ESTADO_MEM_SUCCESS = 9
ESTADO_MEM_FAIL = 10
ESTADO_GAME_OVER = 11

estado_actual = ESTADO_READY 

# ==========================================
# VARIABLES DE CONTROL
# ==========================================
tiempo_inicio = 0
tiempo_limite = 2000
desafios_jugados = 0
max_desafios = 10
aciertos = 0
errores = 0
max_errores = 2
tiempo_reaccion = 0
boton_esperado = -1
error_por_tiempo = False

# Variables de Memoria
MAX_NIVELES_MEMORIA = 5
secuencia_memoria = [0] * MAX_NIVELES_MEMORIA
nivel_actual_memoria = 1
paso_usuario = 0
tiempo_display = 0
indice_display = 0
led_encendido = False

# Antirrebote (Debounce)
ultimo_tiempo_boton = 0
TIEMPO_DEBOUNCE = 250

# ==========================================
# CONFIGURACIÓN DE PINES
# ==========================================
buzzer = PWM(Pin(21))
buzzer.duty(0)

sensor_fsr = ADC(Pin(32))
sensor_fsr.atten(ADC.ATTN_11DB)

# Añadido el Pin 26 para el cuarto LED
leds = [Pin(4, Pin.OUT), Pin(16, Pin.OUT), Pin(17, Pin.OUT), Pin(26, Pin.OUT)]
# Añadido el Pin 23 para el cuarto botón
botones = [Pin(18, Pin.IN, Pin.PULL_UP), Pin(19, Pin.IN, Pin.PULL_UP), Pin(22, Pin.IN, Pin.PULL_UP), Pin(23, Pin.IN, Pin.PULL_UP)]
btn_start = Pin(5, Pin.IN, Pin.PULL_UP)

# Filtro ADC
muestras = [0] * 10
indice_muestra = 0
suma_muestras = 0

# ==========================================
# FUNCIONES AUXILIARES
# ==========================================
def leer_sensor():
    global indice_muestra, suma_muestras
    suma_muestras -= muestras[indice_muestra]
    muestras[indice_muestra] = sensor_fsr.read()
    suma_muestras += muestras[indice_muestra]
    indice_muestra = (indice_muestra + 1) % 10
    return suma_muestras // 10

def emitir_tono(frecuencia, duracion_ms):
    if frecuencia > 0:
        buzzer.freq(frecuencia)
        buzzer.duty(512)
    else:
        buzzer.duty(0)
    time.sleep_ms(duracion_ms)
    buzzer.duty(0)

# ==========================================
# INTERRUPCIONES (ISR)
# ==========================================
def isr_start(p):
    global estado_actual
    if estado_actual == ESTADO_IDLE:
        estado_actual = ESTADO_READY

def procesar_reaccion(indice_boton):
    global estado_actual, tiempo_reaccion, aciertos, errores, error_por_tiempo
    global paso_usuario, nivel_actual_memoria, ultimo_tiempo_boton
    
    tiempo_actual = time.ticks_ms()
    
    if time.ticks_diff(tiempo_actual, ultimo_tiempo_boton) < TIEMPO_DEBOUNCE:
        return
    ultimo_tiempo_boton = tiempo_actual

    if estado_actual == ESTADO_WAITING_REACTION:
        tiempo_reaccion = time.ticks_diff(tiempo_actual, tiempo_inicio)
        if indice_boton == boton_esperado:
            aciertos += 1
            estado_actual = ESTADO_HIT
        else:
            errores += 1
            error_por_tiempo = False
            estado_actual = ESTADO_MISS
            
    elif estado_actual == ESTADO_MEM_WAITING_INPUT:
        if indice_boton == secuencia_memoria[paso_usuario]:
            paso_usuario += 1
            if paso_usuario >= nivel_actual_memoria:
                estado_actual = ESTADO_MEM_SUCCESS
        else:
            estado_actual = ESTADO_MEM_FAIL

btn_start.irq(trigger=Pin.IRQ_FALLING, handler=isr_start)
botones[0].irq(trigger=Pin.IRQ_FALLING, handler=lambda p: procesar_reaccion(0))
botones[1].irq(trigger=Pin.IRQ_FALLING, handler=lambda p: procesar_reaccion(1))
botones[2].irq(trigger=Pin.IRQ_FALLING, handler=lambda p: procesar_reaccion(2))
botones[3].irq(trigger=Pin.IRQ_FALLING, handler=lambda p: procesar_reaccion(3)) # Interrupción para el 4to botón

# ==========================================
# BUCLE PRINCIPAL (FSM)
# ==========================================
print("\nS.M.A.R.T. Iniciado. ¡Arrancando automáticamente!")

poll_obj = select.poll()
poll_obj.register(sys.stdin, select.POLLIN)

while True:
    tension_actual = leer_sensor()
    
    poll_results = poll_obj.poll(0)
    if poll_results and estado_actual == ESTADO_IDLE:
        entrada = sys.stdin.readline().strip()
        if entrada.isdigit():
            nuevo_tiempo = int(entrada)
            if nuevo_tiempo > 500:
                tiempo_limite = nuevo_tiempo
                print(f"IA ajustó dificultad. Nuevo límite (ms): {tiempo_limite}")

    if estado_actual == ESTADO_IDLE:
        pass
        
    elif estado_actual == ESTADO_READY:
        print("\n¡Preparando el sistema! 3...")
        time.sleep(1)
        print("2...")
        time.sleep(1)
        print("1...")
        time.sleep(1)
        desafios_jugados = 0
        aciertos = 0
        errores = 0
        estado_actual = ESTADO_NEW_CHALLENGE
        
    elif estado_actual == ESTADO_NEW_CHALLENGE:
        desafios_jugados += 1
        print(f"\n--- Desafío QTE {desafios_jugados} ---")
        time.sleep(random.uniform(1.5, 4.0))
        
        # Ahora selecciona entre 4 opciones (0, 1, 2, 3)
        boton_esperado = random.randint(0, 3) 
        leds[boton_esperado].value(1)
        
        emitir_tono(1000, 150)
        tiempo_inicio = time.ticks_ms()
        error_por_tiempo = True
        estado_actual = ESTADO_WAITING_REACTION
        
    elif estado_actual == ESTADO_WAITING_REACTION:
        if time.ticks_diff(time.ticks_ms(), tiempo_inicio) > tiempo_limite:
            print("¡Tiempo agotado! (Error)")
            errores += 1
            estado_actual = ESTADO_MISS
            
    elif estado_actual == ESTADO_HIT:
        for led in leds: led.value(0)
        print(">> HIT registrado <<")
        print(f"Tiempo de reaccion: {tiempo_reaccion} ms")
        print(f"Tension fisica (FSR): {tension_actual}")
        time.sleep(1)
        
        if desafios_jugados >= max_desafios:
            print("\n¡Fase de reacción completada!")
            time.sleep(1)
            estado_actual = ESTADO_MEM_INIT
        else:
            estado_actual = ESTADO_NEW_CHALLENGE
            
    elif estado_actual == ESTADO_MISS:
        for led in leds: led.value(0)
        if not error_por_tiempo:
            print(">> ERROR: Botón equivocado <<")
            
        print(f">> MISS registrado << (Errores: {errores}/{max_errores})")
        print(f"Tension fisica al fallar: {tension_actual}")
        emitir_tono(300, 500)
        time.sleep(1)
        
        if errores >= max_errores:
            print("¡Límite de errores alcanzado! Abortando secuencia...")
            estado_actual = ESTADO_GAME_OVER
        elif desafios_jugados >= max_desafios:
            print("\n¡Fase de reacción completada con algunos tropiezos!")
            time.sleep(1)
            estado_actual = ESTADO_MEM_INIT
        else:
            estado_actual = ESTADO_NEW_CHALLENGE
            
    elif estado_actual == ESTADO_MEM_INIT:
        print("\n--- INICIANDO FASE DE MEMORIA ---")
        for i in range(MAX_NIVELES_MEMORIA):
            # Ahora genera secuencias incluyendo el 4to botón
            secuencia_memoria[i] = random.randint(0, 3) 
        nivel_actual_memoria = 1
        indice_display = 0
        led_encendido = False
        estado_actual = ESTADO_MEM_SHOW_SEQUENCE
        tiempo_display = time.ticks_ms()
        
    elif estado_actual == ESTADO_MEM_SHOW_SEQUENCE:
        if time.ticks_diff(time.ticks_ms(), tiempo_display) >= 500:
            tiempo_display = time.ticks_ms()
            
            if not led_encendido:
                leds[secuencia_memoria[indice_display]].value(1)
                led_encendido = True
            else:
                leds[secuencia_memoria[indice_display]].value(0)
                led_encendido = False
                indice_display += 1
                
                if indice_display >= nivel_actual_memoria:
                    paso_usuario = 0
                    print("¡Tu turno!")
                    estado_actual = ESTADO_MEM_WAITING_INPUT
                    
    elif estado_actual == ESTADO_MEM_WAITING_INPUT:
        pass
        
    elif estado_actual == ESTADO_MEM_SUCCESS:
        print("¡Nivel superado!")
        time.sleep(1)
        nivel_actual_memoria += 1
        
        if nivel_actual_memoria > MAX_NIVELES_MEMORIA:
            print("¡PRUEBA DE MEMORIA COMPLETADA CON ÉXITO!")
            estado_actual = ESTADO_GAME_OVER
        else:
            indice_display = 0
            led_encendido = False
            estado_actual = ESTADO_MEM_SHOW_SEQUENCE
            tiempo_display = time.ticks_ms()
            
    elif estado_actual == ESTADO_MEM_FAIL:
        print("¡Secuencia incorrecta! Fallaste la prueba de memoria.")
        for led in leds: led.value(0)
        estado_actual = ESTADO_GAME_OVER
        
    elif estado_actual == ESTADO_GAME_OVER:
        print("\n=== RESULTADOS FINALES DE LA ESTACIÓN ===")
        print("--- FASE REACCIÓN ---")
        print(f"Aciertos: {aciertos}")
        print(f"Errores totales: {errores}")
        print(f"Desafíos superados: {desafios_jugados} de {max_desafios}")
        
        print("--- FASE MEMORIA ---")
        if nivel_actual_memoria > 1 or estado_actual == ESTADO_MEM_FAIL:
            nivel_alcanzado = MAX_NIVELES_MEMORIA if nivel_actual_memoria > MAX_NIVELES_MEMORIA else nivel_actual_memoria - 1
            print(f"Nivel máximo alcanzado: {nivel_alcanzado} de {MAX_NIVELES_MEMORIA}")
        else:
            print("No logró llegar a esta fase.")
            
        print("\n[FIN] Detén y vuelve a correr el script de VS Code para jugar de nuevo.")
        time.sleep(2)
        estado_actual = ESTADO_IDLE
        
    time.sleep(0.05)