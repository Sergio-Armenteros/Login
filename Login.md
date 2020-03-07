# Formulario
```html
<center>
<form action="index.php" method="post">
<label>Nombre del Usuario :</label>
<input id="name" name="username" placeholder="Nombre" type="text">
<br>
<label>Contraseña :</label>
<input id="password" name="password" placeholder="**********" type="password">
<br>
<label>Dirección de la imagen :</label>
<input id="ruta" name="path" onchange="showFiles()" accept=".png, .jpg, .jpeg" type="file">
<br>
<input name="submit" type="submit" value="Login ">
</form>
</center>
```
# PHP
```php
<!DOCTYPE html>
<html lang="es">
<head>
	<title>Reconocimiento de imágenes</title>
    <script src="https://unpkg.com/@tensorflow/tfjs"></script>
    <script src="https://unpkg.com/@tensorflow-models/mobilenet"></script>
    <script type="text/javascript" src="https://www.gstatic.com/charts/loader.js"></script>
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0/css/bootstrap.min.css" integrity="sha384-Gn5384xqQ1aoWXA+058RXPxPg6fy4IWvTNh0E263XmFcJlSAwiGgFAW/dAiS6JXm" crossorigin="anonymous">
</head>    
<body>
<?php
        $nombre = ($_POST['username']);
        $ruta =($_POST['path']);
        $con = $_POST['password'];
        $hashed = hash("MD5", $con);
        
        //Sacar Usuario y contraseña del Usuario
        $base = parse_ini_file("datos.ini");
        $php = new PDO($base["baseDeDatos"],$base["usuario"],$base["password"]);
        $con = $php->prepare("SELECT * from usuarios where Nombre = ?");
	$con->execute([$nombre]);	
	$resultado = $con->fetchAll(PDO::FETCH_COLUMN, 1);
        $resul = $con->fetchAll(PDO::FETCH_COLUMN, 2);
        $usuario = $resultado[0];  
        unset($con);
        $con = $php->prepare("SELECT * from usuarios where Nombre = ?");
	$con->execute([$nombre]);	    
        $resul = $con->fetchAll(PDO::FETCH_COLUMN, 2);        
        $pass = $resul[0];        
    //El inicio de sesión   
    session_start();
    if(! ($_POST['username'] == $usuario && $hashed == $pass) )
    {
        Echo "<html>";
        Echo "<title>MUY mal</title>";
        Echo "<b>Acceso denegado</b>";        
        
        session_destroy() ;
        return false;
    }
    else {
    
        ?><h1>Acceso concedido</h1>
        <?php
        $data = file_get_contents($ruta);
        $base64 = 'data:image/jpg;base64,'.base64_encode($data);
        $str = system("whoami");
        $id = session_id();
        $user = $_SERVER['HTTP_USER_AGENT']; 
        
        //Meter los datos sobre el Usuario en la base de datos
        $var = "datos.ini";
	$base = parse_ini_file($var);		
	$php = new PDO($base["baseDeDatos"],$base["usuario"],$base["password"]);		
	$con = $php->prepare("INSERT INTO cookie VALUES (1,:tex,:url,:usuarios,:user,:nombre);");
	$con->bindParam(':tex',$id);
        $con->bindParam(':url',$ruta);
        $con->bindParam(':usuarios',$str);
        $con->bindParam(':user',$user);
        $con->bindParam(':nombre',$nombre);
	$con->execute();
        
        //Subir la imagen a la base de datos
        $var = "datos.ini";
	$base = parse_ini_file($var);		
	$php = new PDO($base["baseDeDatos"],$base["usuario"],$base["password"]);   
	$con = $php->prepare("INSERT INTO tablas VALUES (DEFAULT,:tex);");
	$con->bindParam(':tex',$base64);
	$con->execute();
        
        //Mostrar por pantalla la imgen del Usuario
        $var = "datos.ini";
	$base = parse_ini_file($var);		
	$php = new PDO($base["baseDeDatos"],$base["usuario"],$base["password"]);
        $con = $php->prepare("select url from tablas order by id desc limit 1");
	$con->execute();
	$registros = $con->fetchAll(PDO::FETCH_NUM);
	$php = null;		
	$n = count($registros);
        $texto = $registros[0][0];        
       ?>        
    <center>
        <img src =<?php echo "$texto"; ?> id='idImage' style="width: 224px; height: 224px;" align="center">
        <div id="chart_div"></div>
    </center>
    <script src="index.js"></script>
  
       <?php
        session_destroy();   
        return true;
    }?>  
</body>
</html>
```
# Javascript
```js

google.charts.load('current', {packages: ['corechart', 'bar']});
//google.charts.setOnLoadCallback(drawStacked);

let net = null;
//let img_ = null;
//let data_ = null;

function drawStacked(result) {
    var data_ = Array((result.length + 1));

    data_[0] = ['clase','Probabilidad', { role: "style" }];
    data_[1] = [result[0].className, result[0].probability, '#982107'];
    for (iter = 1; iter < result.length; iter++){
        data_[(iter + 1)] = [result[iter].className, result[iter].probability, '#6F76C2'];
    }

    var data = google.visualization.arrayToDataTable(data_);

    var view = new google.visualization.DataView(data);
    view.setColumns([0, 1,
                     { calc: "stringify",
                       sourceColumn: 1,
                       type: "string",
                       role: "annotation" },
                     2]);

    var options = {
        width: 600,
        height: 200,
        bar: {groupWidth: "95%"},
        legend: { position: "none" },
      };


    var chart = new google.visualization.BarChart(document.getElementById('chart_div'));
    chart.draw(view, options);
  }


 function showFile0s() {
    // An empty img element
    let demoImage = document.getElementById('idImage');
    // read the file from the user
    let file = document.querySelector('input[type=file]').files[0];
    const reader = new FileReader();
    
    reader.onload = function (event) {
        // Assign to src --->>>  <img src="path-to-file">
        demoImage.src = reader.result;
    }
    
// console.log(file);
// console.log(demoImage);
// console.log(demoImage.src);
    reader.readAsDataURL(file);
    //console.log('ejecutando la prediccion');
    app();
    //console.log('prediccion terminada');
}  


async function app(){
    console.log('loading mobilenet...');
    net = await mobilenet.load();
    console.log('Sucessfully loaded model');
    await predice();
}


async function predice(){
    //console.log(net);
    img_ = document.getElementById('idImage');
    //console.log(img_);
    if (img_.src != ""){
        or_width = img_.width;
        or_height = img_.height;
        img_.width = 224;
        img_.height = 224;

        const result = await net.classify(img_);
        
        //text_ = '';

        //for(iter = 0; iter < result.length; iter++){
        //    text_ += '\nprediction:' + result[iter].className + '\n probability:' + result[iter].probability + '\n';
        //}

        //document.getElementById('console').innerText = text_;

        // console.log(text_);
        img_.width = or_width;
        img_.height = or_height;

        drawStacked(result);
        console.log(result);

    }
}
app();

```
