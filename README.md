# Coordina el registro de clientes, servicios y reservas del sistema.
# Los modelos representan los objetos que administra esta fachada.
from modelos.cliente import Cliente
from modelos.servicio import Servicio
from modelos.reserva import Reserva
# Las excepciones de dominio permiten registrar y propagar errores esperados.
from excepciones.excepciones_personalizadas import (
    SoftwareFJError,
    ClienteNoEncontradoError,
    DatosInvalidosError,
)
# Logger guarda los eventos relevantes de las operaciones.
from utilidades.logger import Logger


class SistemaFJ:
    """Gestiona las colecciones del sistema y coordina las operaciones del dominio."""

    def __init__(self, logger=None):
        """Crea las colecciones internas y configura el registrador de eventos."""
        # Mantiene los clientes registrados durante la ejecución.
        self._clientes = []
        # Mantiene los servicios que se pueden reservar.
        self._servicios = []
        # Mantiene las reservas creadas y su estado actual.
        self._reservas = []
        # Reutiliza el logger proporcionado o crea uno con la configuración estándar.
        self._logger = logger or Logger()

    @property
    def clientes(self):
        """Devuelve una copia de la colección de clientes para evitar cambios externos."""
        return list(self._clientes)

    @property
    def servicios(self):
        """Devuelve una copia de la colección de servicios registrados."""
        return list(self._servicios)

    @property
    def reservas(self):
        """Devuelve una copia de la colección de reservas creadas."""
        return list(self._reservas)

    def registrar_cliente(self, cedula, nombre, email, telefono):
        """Valida, registra y devuelve un cliente nuevo."""
        try:
            # Evita duplicados comparando la cédula normalizada.
            if any(c.cedula == str(cedula).strip() for c in self._clientes):
                raise DatosInvalidosError(f"Cliente con cedula {cedula} ya existe.")
            # El constructor Cliente valida y normaliza los datos suministrados.
            cliente = Cliente(cedula, nombre, email, telefono)
            # Añade el objeto únicamente después de que su creación fue exitosa.
            self._clientes.append(cliente)
            # Registra el resultado satisfactorio para trazabilidad.
            self._logger.info(f"Cliente registrado: {cliente.describir()}")
            return cliente
        except SoftwareFJError as error:
            # Registra el error y vuelve a propagarlo para que lo gestione quien llamó.
            self._logger.error(f"Error registrar cliente ({cedula}): {error}")
            raise

    def buscar_cliente(self, cedula):
        """Busca un cliente por cédula y genera un error si no existe."""
        # Recorre los clientes y compara la identificación sin espacios externos.
        for c in self._clientes:
            if c.cedula == str(cedula).strip():
                return c
        # La búsqueda fallida se expresa mediante una excepción específica.
        raise ClienteNoEncontradoError(f"No se encontro cliente {cedula}.")

    def registrar_servicio(self, servicio):
        """Valida que el servicio sea único y lo incorpora al sistema."""
        try:
            # Solo pueden registrarse instancias de la clase Servicio o sus subclases.
            if not isinstance(servicio, Servicio):
                raise DatosInvalidosError("Servicio invalido.")
            # Impide registrar dos servicios con el mismo identificador.
            if any(s.identificador == servicio.identificador for s in self._servicios):
                raise DatosInvalidosError(f"Servicio {servicio.identificador} ya existe.")
            # Conserva el servicio validado para búsquedas y reservas posteriores.
            self._servicios.append(servicio)
            # Registra la descripción del servicio añadido.
            self._logger.info(f"Servicio registrado: {servicio.describir()}")
            return servicio
        except SoftwareFJError as error:
            # Registra los fallos de dominio y los propaga sin ocultarlos.
            self._logger.error(f"Error registrar servicio: {error}")
            raise

    def buscar_servicio(self, codigo):
        """Busca un servicio por código y reporta si el código no está registrado."""
        # Compara códigos normalizados y devuelve la primera coincidencia.
        for s in self._servicios:
            if s.identificador == str(codigo).strip():
                return s
        # Usa el error de datos inválidos para comunicar un código inexistente.
        raise DatosInvalidosError(f"No se encontro servicio {codigo}.")

    def crear_reserva(self, cedula, codigo_servicio, duracion):
        """Localiza cliente y servicio, crea la reserva y la registra."""
        try:
            # La reserva solo se crea si el cliente ya está registrado.
            cliente = self.buscar_cliente(cedula)
            # También se exige que el servicio exista.
            servicio = self.buscar_servicio(codigo_servicio)
            # El constructor Reserva valida referencias y duración.
            reserva = Reserva(cliente, servicio, duracion)
            # Añade la reserva válida a la colección administrada.
            self._reservas.append(reserva)
            # Registra la creación para auditoría operativa.
            self._logger.info(f"Reserva creada: {reserva.describir()}")
            return reserva
        except SoftwareFJError as error:
            # Registra errores de dominio y permite que el llamador los gestione.
            self._logger.error(f"Error crear reserva: {error}")
            raise

    def confirmar_reserva(self, reserva):
        """Confirma una reserva y registra el resultado."""
        try:
            # Reserva valida estado y disponibilidad antes de confirmarse.
            reserva.confirmar()
            # Incluye la cédula del titular en el registro de confirmación.
            self._logger.info(f"Reserva confirmada: {reserva.cliente.cedula}")
        except SoftwareFJError as error:
            # Registra y propaga cualquier fallo previsto de la confirmación.
            self._logger.error(f"Error confirmar: {error}")
            raise

    def procesar_reserva(self, reserva, impuesto=0.0, descuento=0.0):
        """Calcula el costo de una reserva y devuelve el importe resultante."""
        try:
            # Delegar en Reserva conserva allí las reglas de estado y cálculo.
            costo = reserva.procesar(impuesto=impuesto, descuento=descuento)
            # Registra el valor calculado para seguimiento.
            self._logger.info(f"Reserva procesada: costo=${costo}")
            return costo
        except SoftwareFJError as error:
            # Registra y propaga los errores de procesamiento.
            self._logger.error(f"Error procesar: {error}")
            raise

    def cancelar_reserva(self, reserva):
        """Cancela una reserva válida y registra la operación."""
        try:
            # Reserva comprueba si el estado actual permite la cancelación.
            reserva.cancelar()
            # Registra la identificación del cliente titular.
            self._logger.info(f"Reserva cancelada: {reserva.cliente.cedula}")
        except SoftwareFJError as error:
            # Registra y propaga los errores de cancelación.
            self._logger.error(f"Error cancelar: {error}")
            raise

    def resumen(self):
        """Devuelve un resumen de cantidades para cada tipo de objeto administrado."""
        return (f"=== RESUMEN SOFTWARE FJ ===\n"
                f"Clientes : {len(self._clientes)}\n"
                f"Servicios: {len(self._servicios)}\n"
                f"Reservas : {len(self._reservas)}\n"
                f"===========================")
