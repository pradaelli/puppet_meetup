# puppet_meetup
Het doel van deze demo is om vanuit de module-makers een eerste indruk te krijgen van hoe een module gemaakt kan worden en de werking ervan. Het gaat om een simpele module, maar wel volledig uitgewerkt inclusies hoe de module ge-applied moet worden.<br />
In deze demo gaan we een module maken die het systeem voorbereid om het maken van virtual hosts doormiddel van httpd.<br />
Om dit te doen hebben we de volgende elementen nodig:
* httpd packahe installeren
* httpd service managen
* Virtual hosts configureren

Beginnen met het opstellen van een manifest die de httpd package installeert. Hiervoor controlleren we eerst dat httpd niet aanwezig is:
```
which httpd
```
Should return "not found".<br />
Maar in je workingdirectory een package.pp file aan (we zullen vanaf nu de puppetfiles "manifests" noemen) met de volgende inhoud:
```
package { 'httpdpackage':
  ensure => present,
  name   => 'httpd',
}
```
Hier is "package" een resource dat de packages kan managen en "ensure" en "name" zijn twee atributen ervan.<br />
Laat dit manifest nu runnen met het comando:
```
sudo puppet apply package.pp
```
en controlleer dat httpd weer aanwezig is.<br />

Nu maken we een manifest aan om de service te laten runnen/stoppen. Maak hiervoor een service.pp manifest met de volgende inhoud aan:
```
service { 'httpdservice':
  ensure => 'running',
  name   => 'httpd',
  enable => true,
}
```
Run dit met:
```
sudo puppet apply service.pp
```
Bekijk de status van de service met:
```
sudo service httpd status
```
Probeer de service handmatig te stoppen en dan weer puppet te runnen en de status opnieuw te bekijken.<br />
We merken nu al dat het handig is om de twee manifest te combineren (zoals we dat wel vaker doen in programmeertalen) in subclassen en deze dan aanroepen vanaf een hoofdclasse. Dit is het begin van onze puppet-module. De filestructuur van een puppetmodule is als voglt:
modulename<br />
⋅⋅manifests<br />
⋅⋅⋅⋅init.pp<br />
⋅⋅⋅⋅subclass1.pp<br />
⋅⋅⋅⋅subclass2.pp<br />

De hoofdclasse waar puppet default naar kijkt heeft normaal gesproken de naam van de module zelf en is als default in de init.pp file te vinden.<br />
Maak voor onze demo een apache/manifests/init.pp file aan met de volgende inhoud:
```
class apache {
  include ::apache::package#aanroepen van de package subclasse
  include ::apache::package#aanroepen van de service subclasse
}
```
en plaats maak een package en service subclasse aan op de zelfde manier:<br />
apache/manifests/package.pp
```
class apache::package {
  package { 'httpdpackage':
    ensure => 'present',
    name   => 'httpdi',
  }
}
```
apache/manifests/service.pp
```
class apache::service {
  service { 'httpdservice':
    ensure  => 'running',
    name    => 'httpd',
    enable  => true,
    require => Package['httpdpackage'],#we hebben de package nodig om de service te runnen
  }
}
```
Vervolgens kunnen we de hele module runnen met:
```
sudo puppet apply -e "include ::apache" --modulepath=.
```
en bekijke het resultaat.<br />
Over het algemeen is het erg handig om met parameters te werken. Deze parameters kunnen zowel in de classe zelf geset worden, als vanuit een andere classe maar ook vanuit hiera (zoals we het binnen kpn veel doen). Laten we de vorige classe aanpassen en parameters introduceren. Ik zal ook vast de service classe aanpassen zodat het de httpd.conf file aanpast om later de Vhost te configureren.<br />
init.pp:
```
class apache {
  class{ '::apache::package'} #aanroepen van de package subclasse
  class{ '::apache::package'} #aanroepen van de service subclasse
}
```
ik ga dit even overlaan vanwege de tijd...<br />


add the following resource in service.pp:
```
file_line { 'httpdline':
  ensure => present,
  path   => "/etc/httpd/conf/httpd.conf",
  line   => "NameVirtualHost *:80",   ###*
  notify => Service['httpdservice'] ###als deze resource wordt uitgevoert dan zal de service ook worden aangeroepen ###
}
```

Wat nu nog gedaan moet worden is het configureren van onze virtual hosts. Hiervoor maken we een nieuwe subclasse aan met de naam vhost.pp en de volgende inhoud:
```
define apache::vhost {  ######let op dat dit een define typy is! dit soort classes kun je vaker aanroepen

  file { "/etc/httpd/conf.d/$name.conf": ###configure the virtualhost on port 80
    ensure  => present,
    content => "
<VirtualHost *:80>
ServerName $name
DocumentRoot /var/www/html/$name
</VirtualHost>"
  }

  file { "/var/www/html/$name":####zet de inhoud van de virtual host
    ensure => directory,
    owner  => 'apache',
    group  => 'apache',
    mode   => '755',
  }

  file { "/var/www/html/$name/index.html":
    ensure  => present,
    content => '
<!DOCTYPE html PUBLIC "-//IETF//DTD HTML 2.0//EN">
<HTML>
   <HEAD>
      <TITLE>
         A Small Hello 
      </TITLE>
   </HEAD>
<BODY>
   <H1>Hi</H1>
   <P>This is my virtual host managed by puppet with the name $name.</P> 
</BODY>
</HTML>',
    owner   => 'apache',
    group   => 'apache',
    mode    => '644',
  }
}
```
en we passen de init.pp als volgt aan om dize classe aan te roepen:
```
class apache {
  class { '::apache::package':}
  class { '::apache::service':}
  apache::vhost { 'myhost':   ### de virtual host met de naam "myhost" is hier aangemaakt ###
    notify => Service['httpdservice'] ###wanneer je een virtual host aanmaakt wil je de service weer op starten ###
  }
}
```
Run dit script en bekijk het resultaat aan de clientside op http://myhost!! (op deze machine moet je wel eerst de hosts tables aanpassen)<br />
Het is nu erg makkelijk om een tweede virtual host aan te maken en zo nog 100.
































