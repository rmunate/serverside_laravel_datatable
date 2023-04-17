# ServerSide Laravel
## _Codigo ServerSide_

1. Crear una Ruta Get:
```php
//Aunque es ruta get no pondremos variables en su base {...}, ya que necesitamos que todo ingrese por Request.
Route::get('/module/index-table', [NameController::class, 'indexTable']);
```

2. Metodo Server Side
```php
//Metodo en el controlador de Laravel
public function indexTable(Request $request){+
    #El Request contendra unos datos que envia JS al BackEnd

    # Recursos del Servidor
    # Esto se configura para que no haya un limite de ejecucion ni un maximo de memoria empleada para ejecutar el Script.
    ini_set('max_execution_time', 0);
    ini_set('memory_limit', '-1');

    # Columnas
    # Aqui debe estar la totalidad de columnas que se van a imprimir en el HTML.
    # Importante que el arreglo tenga los index en orden, como se puede ver a continuacion inicia en 0 y termina en 6 sin saltar ninguno.
    # Estos nombres los usará JS para los procesos de impresión, ordenación y filtro.
    $columns = array(
        0 => 'nombre',
        1 => 'interes',
        2 => 'min_valor',
        3 => 'max_valor',
        4 => 'min_duracion',
        5 => 'max_duracion',
        6 => 'action',
    );

    # Consulta SQL Preparada
    # Se recomienda para efectos del serverside siempre trabajar con Query Builder, ya que necesitamos campos con alias claros y especificos para los filtros, se puede trabajar con Eloquent sin embargo no se relaciona en el manual.
    # Si nos fijamos los alias de ciertos campos tienen el mismo nombre del arreglo que esta al inicio de este metodo, esto para que sea claro para el sistema el campo relacionado.
    $consulta = DB::table('admin_lineas')
    ->join('admin_empresas', 'admin_lineas.empresa_id', '=', 'admin_empresas.id')
        ->select(
            'admin_lineas.id AS id',
            'admin_lineas.nombre AS nombre',
            'admin_lineas.interes AS interes',
            'admin_lineas.min_valor AS min_valor',
            'admin_lineas.max_valor AS max_valor',
            'admin_lineas.min_duracion AS min_duracion',
            'admin_lineas.max_duracion AS max_duracion',
            'admin_empresas.id AS empresa_id',
            'admin_empresas.nombre AS empresa',
        );

    # Informacion para Datatable
    # En este bloque se extrae el total de registro de la base de datos que se mostrara en el pie de la tabla
    # Se ejecuta el metodo count de la consulta sin ejecutar.
    $totalData = $consulta->count();

    # Este sera el limite de datos, por defecto es 10, pero esta atado a la lista desplegable de la tabla
    $limit = $request->input('length');

    # Esto define cual es el registro inicio de la consulta.
    $start = $request->input('start');

    # Esto define el orden de los campos.
    $order = $columns[$request->input('order.0.column')];
    $dir = $request->input('order.0.dir');

    if (empty($request->input('search.value'))) {

        # Por defecto ingresa por aca ya que no hay filtros en la DataTable.
        $posts = $consulta->offset($start)->limit($limit)->orderBy($order, $dir)->get();
        $totalFiltered = $totalData;

    } else {

        # Ingresa por aqui cuando hay filtros en la tabla.
        
        # Esta variable contiene el valor del Filtro
        $search = $request->input('search.value');

        # Desde aqui se aplican los filtros por like a cada uno de los valores de la tabla.
        # Si algun campo es id como el estado, hay que construir un helper para remplzar el texto por ID en la busqueda
        # Desde la segunda condicional debe ser orWhere, para que se cumpla cualquiera.
        # Si se deben asociar varias condiciones en la busqueda, trate de usar un ->where(function($query))
        $clausulas = $consulta->where('admin_lineas.nombre', 'like', "%{$search}%");
        $clausulas = $consulta->orWhere('admin_lineas.interes', 'like', "%{$search}%");
        $clausulas = $consulta->orWhere('admin_lineas.min_valor', 'like', "%{$search}%");
        $clausulas = $consulta->orWhere('admin_lineas.max_valor', 'like', "%{$search}%");
        $clausulas = $consulta->orWhere('admin_lineas.min_duracion', 'like', "%{$search}%");
        $clausulas = $consulta->orWhere('admin_lineas.max_duracion', 'like', "%{$search}%");

        # Ejecucion de la cuenta y los limites de busqueda
        $totalFiltered = $clausulas->count();
        $posts = $clausulas->offset($start)->limit($limit)->orderBy($order, $dir)->get();
    }

    # Aqui se consultaran los permisos del usuario para renderizar los  botones en el Front
    $editar = Auth::user()->can('modulo.editar');
    $eliminar = Auth::user()->can('modulo.eliminar');

    # Este arreglo debe ser llenado con los datos de la tabla
    $data = array();

    # Ejecutar solamente de haber resultados.
    if ($posts) {
        # Iterar la Consulta.
        foreach ($posts as $r) {
            # En esta itaracion el index del array debe ser el mismo del alias y del arreglo delas columnas iniciales.
            $nestedData['nombre'] = $r->nombre;
            $nestedData['interes'] = $r->interes;
            $nestedData['min_valor'] = currencyFormat($r->min_valor);
            $nestedData['max_valor'] = currencyFormat($r->max_valor);
            $nestedData['min_duracion'] = $r->min_duracion;
            $nestedData['max_duracion'] = $r->max_duracion;
            $nestedData['action'] = [
                # Los valores para la columna acciones deben agregarsen en este lugar.
                'id' => Crypt::encrypt($r->id),
                'registro' => [
                    'id' => $r->id,
                    'nombre' => $r->nombre,
                    'interes' => $r->interes,
                    'min_valor' => $r->min_valor,
                    'max_valor' => $r->max_valor,
                    'min_duracion' => $r->min_duracion,
                    'max_duracion' => $r->max_duracion,
                    'empresa_id' => $r->empresa_id,
                ],
                'editar' => $editar
            ];
            $data[] = $nestedData;
        }
    }

    # Formato de Retorno 
    $json_data = array(
        "draw" => intval($request->input('draw')),
        "recordsTotal" => intval($totalData),
        "recordsFiltered" => intval($totalFiltered),
        "data" => $data,
    );

    return json_encode($json_data);
}

```

3. Crear Logica con JQUERY
```javascript
//Crear una variable global para alojar la tabla:
var tabla;

//Dentro del bloque de la carga del documento se debe llenar esta variable con la datatable.
$(document).ready(function () {

    //El id dependera del asignado a la tbla HTML
    tabla = $('#table_id_').DataTable({
        "processing": true,
        "serverSide": true,
        "responsive": true,
        //Aqui se debe definir que no se podra ordenar el campo de acciones ya que son botones
        "columnDefs": [
            {"targets": [6],"orderable": false,"searchable": false}
        ],
        //Tipo de paginacion.
        "pagingType": "full_numbers",
        //Peticion AJAX
        //La base URL puede ser llamada desde un Meta o desde una funcion en el Layaut principal de la plantilla.
        //De otro modo con la libreria PHP2JS se empleará el siguiente ejemplo (https://github.com/rmunate/PHP2JS)
        "ajax": {
            "url": __PHP().baseUrl() + '/lineas_de_credito/dataIndex',
            "type": "GET",
            "datatype": "JSON"
        },
        //Aqui se trabajara con las variables recibidas
        //Cada data debe llevar el nombre del array columns del servidor.
        columns: [
            {data: "nombre"},
            {data: "interes"},
            {data: "min_valor"},
            {data: "max_valor"},
            {data: "min_duracion"},
            {data: "max_duracion"},
            {data: "action",
                //En los casos de Renderizar botones o contenidos
                render: function (data,type,full) {
                    //Crear las variables vacias inicialmente para luego llenarlas cuando corresponda.
                    let editar = '';
                    //Aqui si se tiene el permiso se renderiza el boton.
                    if (data.editar) {
                        editar = `<a data-id="${data.registro.id}" data-tipoid="${data.registro.tipo_id}" data-nombre="${data.registro.nombre}" data-interes="${data.registro.interes}" data-minvalor="${data.registro.min_valor}" data-maxvalor="${data.registro.max_valor}" data-minduracion="${data.registro.min_duracion}" data-maxduracion="${data.registro.max_duracion}" data-empresa="${data.registro.empresa_id}" type="button" class="btn btn-sm btn-info btn-editar" title="Ver Detalles"><i class="fa fa-pencil"></i></a>`;
                    }
                    //Siempre se retornan todas las variables, recordemos que de forma inicial estan vacias, por lo cual si no tien permisos, retornada el div vacio.
                    return `<div class="btn-group">${editar}</div>`;
                }
            }
        ],
        //Descargar el lenguaje en elgun lugar de la carpeta publica y llamar el JSON para traducir al español.
        language: {
            url: __PHP().baseUrl() + "/plugins/datatable/lang/es_es.json"
        }
    });



    //Siempre que ejecutemos una accion en la tabla, de crear, editar o inactivar/eliminar, no debemos recargar la pagina, en su lugar 
     tabla.ajax.reload();

})
```

4. HTML:
```html
<!-- Aqui debe estar asignado el id de la tabla a transformar -->
<table id="table-id" class="table table-hover">
    <thead>
        <!-- La cantidad de columnas debe coincidir con la variable $columns y con los datas de JavaScript -->
        <tr class="text-center">
            <th>Nombre</th>
            <th>Interes</th>
            <th>Valor Mínimo</th>
            <th>Valor Máximo</th>
            <th>Duración Mínima</th>
            <th>Duración Máxima</th>
            <th>Acciones</th> 
        </tr>
    </thead>
    <tbody class="text-center bd-tbl"></tbody>
</table>
```
