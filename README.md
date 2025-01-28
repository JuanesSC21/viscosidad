# viscosidad.s
    ! Sección de datos
    .section ".bss"
        pasos:       .word   0         ! Número de pasos
        Kv:          .word   0         ! Constante de viscosidad
        m:           .word   0         ! Masa
        t:           .word   0         ! Delta de tiempo
        Pos_i:       .word   0, 0      ! Posición inicial (x, y)
        V_i:         .word   0, 0      ! Velocidad inicial (Vx, Vy)
        F:           .word   0, 0      ! Fuerza (Fx, Fy)
        V:           .word   0, 0      ! Velocidad actual (Vx, Vy)
        delta_pos:   .word   0, 0      ! Cambio de posición (dx, dy)
        pos_lista:   .skip   400       ! Espacio para almacenar posiciones (200 pasos máx, 2 componentes por paso)

    ! Sección de código
    .section ".text"
    .global main

    main:
        ! Inicialización
        LD    [%lo(V_i)], %o0       ! Cargar Vx inicial en %o0
        ST    %o0, [%lo(V)]         ! Guardar Vx en V[0]
        LD    [%lo(V_i)+4], %o1     ! Cargar Vy inicial en %o1
        ST    %o1, [%lo(V)+4]       ! Guardar Vy en V[1]
        LD    [%lo(Pos_i)], %o2     ! Cargar Pos_x inicial en %o2
        ST    %o2, [%lo(pos_lista)] ! Guardar Pos_x inicial en lista[0]
        LD    [%lo(Pos_i)+4], %o3   ! Cargar Pos_y inicial en %o3
        ST    %o3, [%lo(pos_lista)+4]! Guardar Pos_y inicial en lista[1]
        LD    [%lo(pasos)], %o4     ! Cargar número de pasos en %o4
        LD    [%lo(t)], %o5         ! Cargar delta de tiempo en %o5
        SETHI    %hi(0), %l0          ! Inicializar contador a 0
        SETHI    %hi(pos_lista), %l1  ! Dirección base de la lista

    loop_pasos:
        CMP    %l0, %o4             ! Comparar contador con número de pasos
        BGE    end                  ! Si contador >= pasos, salir del bucle
        NOP
    
        ! Calcular F = -Kv * V
        LD    [%lo(Kv)], %o6       ! Cargar Kv en %o6
        LD    [%lo(V)], %o7        ! Cargar Vx en %o7
        SMUL    %o6, %o7, %o8        ! F_x = Kv * Vx
        NEG    %o8                  ! F_x = -F_x
        ST    %o8, [%lo(F)]        ! Guardar F_x en F[0]
        LD    [%lo(V)+4], %o7      ! Cargar Vy en %o7
        SMUL    %o6, %o7, %o8        ! F_y = Kv * Vy
        NEG    %o8                  ! F_y = -F_y
        ST    %o8, [%lo(F)+4]      ! Guardar F_y en F[1]
    
        ! Calcular a = F / m
        LD    [%lo(m)], %o9        ! Cargar m en %o9
        LD    [%lo(F)], %o10       ! Cargar F_x en %o10
        SDIV    %o10, %o9, %o11      ! a_x = F_x / m
        ST    %o11, [%lo(delta_pos)] ! Guardar a_x en delta_pos[0]
        LD    [%lo(F)+4], %o10     ! Cargar F_y en %o10
        SDIV    %o10, %o9, %o11      ! a_y = F_y / m
        ST    %o11, [%lo(delta_pos)+4] ! Guardar a_y en delta_pos[1]

        ! Calcular nueva velocidad: V = V + a * t
        LD    [%lo(delta_pos)], %o12 ! Cargar a_x en %o12
        SMUL    %o12, %o5, %o12      ! a_x * t
        LD    [%lo(V)], %o13       ! Cargar Vx actual
        ADD    %o13, %o12, %o13     ! Vx = Vx + (a_x * t)
        ST    %o13, [%lo(V)]       ! Guardar nuevo Vx
        LD    [%lo(delta_pos)+4], %o12 ! Cargar a_y en %o12
        SMUL    %o12, %o5, %o12      ! a_y * t
        LD    [%lo(V)+4], %o13     ! Cargar Vy actual
        ADD    %o13, %o12, %o13     ! Vy = Vy + (a_y * t)
        ST    %o13, [%lo(V)+4]     ! Guardar nuevo Vy

        ! Calcular nueva posición: delta_pos = V * t + (a*t*t)/2  
        ! Para el componente X
        SMUL    %o5, %o5, %o14       ! t*t
        LD    [%lo(delta_pos)], %o12 ! Cargar a_x en %o12
        SMUL    %o12, %o14, %o14     ! t*t*a
        LD    [%lo(V)], %o15       ! Cargar Vx
        SMUL    %o15, %o5, %o18      ! dx = Vx * t
        ADD    %o18, %o14, %o14     
        MOV    2, %l2              ! Cargar el divisor (2) en un registro temporal
        SDIV    %o14, %l2, %o14    ! División por 2
        ST    %o14, [%lo(delta_pos)] ! Guardar dx
        
        ! Para el componente y
        SMUL    %o5, %o5, %o14       ! t*t
        LD    [%lo(delta_pos)+4], %o12 ! Cargar a_y en %o12
        SMUL    %o12, %o14, %o14     ! t*t*a
        LD    [%lo(V)+4], %o15       ! Cargar Vy
        SMUL    %o15, %o5, %o18      ! dx = Vy * t
        ADD    %o18, %014, %o14     
        MOV    2, %l2              ! Cargar el divisor (2) en un registro temporal
        SDIV    %o14, %l2, %o14    ! División por 2
        ST    %o14, [%lo(delta_pos)+4] ! Guardar dy
    
        ! Actualizar posición y guardar en lista
        LD    [%l1], %o16          ! Cargar Pos_x actual
        ADD    %o16, %o14, %o16     ! Pos_x = Pos_x + dx
        ST    %o16, [%l1+8]        ! Guardar en lista siguiente posición X
        LD    [%l1+4], %o17        ! Cargar Pos_y actual
        ADD    %o17, %o15, %o17     ! Pos_y = Pos_y + dy
        ST    %o17, [%l1+12]       ! Guardar en lista siguiente posición Y
    
        ADD    %l1, 8, %l1          ! Mover a la siguiente posición en la lista
        INC    %l0                  ! Incrementar contador
        BA    loop_pasos           ! Saltar al inicio del bucle
        NOP
    
    end:
        RETL                       ! Finalizar el programa
        NOP
