//ELIZABETH ROJAS PEREZ
use anchor_lang::prelude::*;
declare_id!("FNjRAE4EVkijUZBBP1Svfm3nBp9JKjTrKq7CkLVu3WN");

// declaracion de constantes
pub const MAX_ESTUDIANTES: usize = 200;
pub const MAX_NOMBRE_ESTUDIANTE: usize = 40;
pub const MAX_GRUPO: usize = 10;
pub const MAX_TUTOR: usize = 60;
pub const MAX_NOMBRE_KINDER: usize = 60;

// Rango de edades
pub const EDAD_MIN: u8 = 3;
pub const EDAD_MAX: u8 = 6;

// iniciamos programa
#[program]
pub mod kinder {
    use super::*;

    /// Crea el registro del kínder (PDA por owner).
    pub fn crear_kinder(ctx: Context<NuevoKinder>, nombre: String) -> Result<()> {
        require!(
            nombre.len() <= MAX_NOMBRE_KINDER,
            KinderError::NombreKinderMuyLargo
        );

        let owner = ctx.accounts.owner.key();
        let estudiantes: Vec<Estudiante> = Vec::new();

        ctx.accounts.kinder.set_inner(Kinder {
            owner,
            nombre,
            estudiantes,
        });
        Ok(())
    }

    /// Crea/agrega un estudiante al kínder.
    pub fn crear_estudiante(
        ctx: Context<EditarKinder>,
        nombre: String,
        edad: u8,
        grupo: String,
        tutor: String,
    ) -> Result<()> {
        // Validaciones de entrada
        require!(edad >= EDAD_MIN && edad <= EDAD_MAX, KinderError::EdadInvalida);
        require!(nombre.len() <= MAX_NOMBRE_ESTUDIANTE, KinderError::NombreMuyLargo);
        require!(grupo.len() <= MAX_GRUPO, KinderError::GrupoMuyLargo);
        require!(tutor.len() <= MAX_TUTOR, KinderError::TutorMuyLargo);

        let kinder = &mut ctx.accounts.kinder;
        require!(
            kinder.estudiantes.len() < MAX_ESTUDIANTES,
            KinderError::ListaLlena
        );

        let estudiante = Estudiante {
            nombre,
            edad,
            grupo,
            tutor,
            activo: true,
        };

        kinder.estudiantes.push(estudiante);
        Ok(())
    }

    /// Actualiza campos de un estudiante por índice (sólo los que envíes en `cambios`).
    pub fn actualizar_estudiante(
        ctx: Context<EditarKinder>,
        idx: u16,
        cambios: EstudianteActualizacion,
    ) -> Result<()> {
        let kinder = &mut ctx.accounts.kinder;
        let len = kinder.estudiantes.len();
        require!((idx as usize) < len, KinderError::IndiceFueraDeRango);

        let e = &mut kinder.estudiantes[idx as usize];

        if let Some(nombre) = cambios.nombre {
            require!(nombre.len() <= MAX_NOMBRE_ESTUDIANTE, KinderError::NombreMuyLargo);
            e.nombre = nombre;
        }
        if let Some(edad) = cambios.edad {
            require!(edad >= EDAD_MIN && edad <= EDAD_MAX, KinderError::EdadInvalida);
            e.edad = edad;
        }
        if let Some(grupo) = cambios.grupo {
            require!(grupo.len() <= MAX_GRUPO, KinderError::GrupoMuyLargo);
            e.grupo = grupo;
        }
        if let Some(tutor) = cambios.tutor {
            require!(tutor.len() <= MAX_TUTOR, KinderError::TutorMuyLargo);
            e.tutor = tutor;
        }
        if let Some(activo) = cambios.activo {
            e.activo = activo;
        }

        Ok(())
    }

    /// Elimina un estudiante por índice (conserva el orden del vector).
    pub fn eliminar_estudiante(ctx: Context<EditarKinder>, idx: u16) -> Result<()> {
        let kinder = &mut ctx.accounts.kinder;
        let len = kinder.estudiantes.len();
        require!((idx as usize) < len, KinderError::IndiceFueraDeRango);

        // Mantener orden (O(n)). Si no te importa el orden, usa swap_remove para O(1).
        kinder.estudiantes.remove(idx as usize);
        Ok(())
    }

    /// Listar (READ). En on-chain lo mostramos por logs; lectura real es off-chain (RPC).
    pub fn listar_estudiantes(ctx: Context<VerKinder>) -> Result<()> {
        let k = &ctx.accounts.kinder;
        msg!("Kínder: {}", k.nombre);
        msg!("Total estudiantes: {}", k.estudiantes.len());
        for (i, e) in k.estudiantes.iter().enumerate() {
            msg!(
                "#{i}: {{ nombre: {}, edad: {}, grupo: {}, tutor: {}, activo: {} }}",
                e.nombre,
                e.edad,
                e.grupo,
                e.tutor,
                e.activo
            );
        }
        Ok(())
    }
}

//CUENTAS

#[account]
#[derive(InitSpace)]
pub struct Kinder {
    pub owner: Pubkey,

    #[max_len(MAX_NOMBRE_KINDER)]
    pub nombre: String,

    #[max_len(MAX_ESTUDIANTES)]
    pub estudiantes: Vec<Estudiante>,
}

#[derive(AnchorSerialize, AnchorDeserialize, Clone, InitSpace, PartialEq, Debug)]
pub struct Estudiante {
    #[max_len(MAX_NOMBRE_ESTUDIANTE)]
    pub nombre: String,
    pub edad: u8,
    #[max_len(MAX_GRUPO)]
    pub grupo: String,
    #[max_len(MAX_TUTOR)]
    pub tutor: String,
    pub activo: bool,
}

/// Estructura para actualizar campos opcionales de un estudiante.
/// (Sólo se aplican los Some(..))
#[derive(AnchorSerialize, AnchorDeserialize, Clone, Debug)]
pub struct EstudianteActualizacion {
    pub nombre: Option<String>,
    pub edad: Option<u8>,
    pub grupo: Option<String>,
    pub tutor: Option<String>,
    pub activo: Option<bool>,
}

// CONTEXTO




#[derive(Accounts)]
pub struct NuevoKinder<'info> {
    #[account(mut)]
    pub owner: Signer<'info>,

    #[account(
        init,
        payer = owner,
        space = 8 + Kinder::INIT_SPACE,
        seeds = [b"kinder", owner.key().as_ref()],
        bump
    )]
    pub kinder: Account<'info, Kinder>,

    pub system_program: Program<'info, System>,
}



#[derive(Accounts)]
pub struct EditarKinder<'info> {
    pub owner: Signer<'info>,

    #[account(
        mut,
        seeds = [b"kinder", owner.key().as_ref()],
        bump,
        has_one = owner
    )]
    pub kinder: Account<'info, Kinder>,
}

#[derive(Accounts)]
pub struct VerKinder<'info> {
    #[account(
        seeds = [b"kinder", kinder.owner.as_ref()],
        bump
    )]
    pub kinder: Account<'info, Kinder>,
}

// errores

#[error_code]
pub enum KinderError {
    #[msg("La lista de estudiantes está llena.")]
    ListaLlena,
    #[msg("Índice fuera de rango.")]
    IndiceFueraDeRango,
    #[msg("Edad inválida para educación preescolar.")]
    EdadInvalida,
    #[msg("El nombre del estudiante excede el máximo permitido.")]
    NombreMuyLargo,
    #[msg("El grupo excede el máximo permitido.")]
    GrupoMuyLargo,
    #[msg("El nombre del tutor excede el máximo permitido.")]
    TutorMuyLargo,
    #[msg("El nombre del kínder excede el máximo permitido.")]
    NombreKinderMuyLargo,
}
