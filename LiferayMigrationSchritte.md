Schritte Migration
===================
#1. Download
##Liferay Portal
lege Ordner liferay-portal-6.2.0-ga1 ins ~/code Verzeichnis

``` bash
mkdir lr_migration
cd lr_migration/
wget http://surfnet.dl.sourceforge.net/project/lportal/Liferay%20Portal/6.2.0%20GA1/liferay-portal-tomcat-6.2.0-ce-ga1-20131101192857659.zip
```

##Liferay Sdk
``` bash
mv liferaySdk liferaySdk.old
wget http://surfnet.dl.sourceforge.net/project/lportal/Liferay%20Portal/6.2.0%20GA1/liferay-plugins-sdk-6.2.0-ce-ga1-20131101192857659.zip
unzip liferay-plugins-sdk-6.2.0-ce-ga1-20131101192857659.zip
mv liferay-plugins-sdk-6.2.0 liferaySdk
```

#2 Install & config
##liferay portal

``` bash
cd code
unzip ../lr_migration/liferay-portal-tomcat-6.2.0-ce-ga1-20131101192857659.zip
cp -r liferay-portal-6.1.0-ce-ga1/data/* liferay-portal-6.2.0-ce-ga1/data/
cp liferay-portal-6.1.0-ce-ga1/portal-* liferay-portal-6.2.0-ce-ga1/
```

###Verändere die portal-*.properties dahingehend, 
dass die Tomcat- und Liferayversionen wieder stimmen. 
Entferne außerdem permissions.user.check.algorithm=6 aus liferay-ext.properties, da veraltet und Server Error wirft (jedoch ohne sichtbare Auswirkungen)

``` bash
vi liferay-portal-6.2.0-ce-ga1/portal-ext.properties

-------------------
hot.deploy.listeners=\
com.liferay.portal.deploy.hot.PluginPackageHotDeployListener,\
com.liferay.portal.deploy.hot.HookHotDeployListener,\
com.liferay.portal.deploy.hot.LayoutTemplateHotDeployListener,\
com.liferay.portal.deploy.hot.PortletHotDeployListener,\
com.liferay.portal.deploy.hot.ThemeHotDeployListener,\
com.liferay.portal.deploy.hot.ThemeLoaderHotDeployListener,\
com.liferay.portal.deploy.hot.MessagingHotDeployListener

# Make sure to be logged in after upgrade
passwords.encryption.algorithm.legacy=SHA
-----------------------

vi liferay-portal-6.2.0-ce-ga1/portal-ide.properties

-------------------------------
auto.deploy.tomcat.conf.dir=/home/politaktiv/code/liferay-portal-6.2.0-ce-ga1/tomcat-7.0.42/conf/Catalina/localhost
------------------------------
```

##Liferay SDK

Ändere build.politaktiv.properties in liferaySdk dahingehend, dass Liferay- und Tomcatversionen stimmen	

``` bash
leafpad liferaySdk/build.politaktiv.properties
```

###Build.props konfigurieren
``` bash
vi liferaySdk/build.properties

=================
javac.compiler=modern
=================
build.politaktiv.properties vom alten liferaySdk rüberkopieren
```

##Remove Not used / working Portlets
Lösche calendar-portlet aus tomcat-7.0.42/webapps, da dieser beim Start fehler wirft (wurde abgesprochen, dass erlaubt)
``` bash
cd ~/code/liferay-portal-6.2.0-ce-ga1/tomcat-7.0.42/webapps
rm -r calendar-portlet
rm -r welcome-theme
```
##Deploy needed portlets
* PA-Theme

#2 Daten Migration

##Erstelle Backup für Datenbank (Snapshot VM)

##Umbennennen: Ordner, der zu Datenbankproblemen beim Update führt:
``` bash
cd ~/code/liferay-portal-6.2.0-ce-ga1/data/document_library/10132/0/wiki/14014
mv Protokoll_20110409_Fachbeirat_verk\?rzt.pdf/ Protokoll_20110409_Fachbeirat_verkuerzt.pdf/
```

##Lösche aus root@localhost/lportal/TABLE/PollsVote das offensichtliche Duplikat, würde zu Fehlern führen
``` bash
/opt/java/DbVisualizer-8.0.8/dbvis
```

``` sql
delete from PollsVote;
```

##Lösche aus root@localhost/lportal/TABLE/DLFileEntry und aus ../DLFileVersion alle Einträge mit mit groupID=11259, würde zu Fehlern führen
``` sql
-> delete from lportal.DLFileEntry where groupid = '11259';
-> delete from lportal.DLFileVersion where groupid = '11259';
```

##MBThreadFlag um doppelte Relationen bereinigen:
``` sql
-- sollte leer sein --------------------------
select * from MBThreadFlag as a, MBThreadFlag as b 
where a.userId = b.userId and a.threadId = b.threadId and a.threadFlagId <> b.threadFlagId;

-- deletes ------------------------------
delete from MBThreadFlag where threadFlagId in (49173, 82011, 82012, 82014, 82017);
```

##ResourceBlock um doppelte Relationen bereinigen:

``` sql
-- sollte leer sein ---------------------------
select * 
from ResourceBlock as a join ResourceBlock as b
        on a.companyId = b.companyId and a.groupId = b.groupId 
        and a.name = b.name and a.permissionsHash = b.permissionsHash
        and a.resourceBlockId <> b.resourceBlockId;

-- deletes ------------------------------
delete from ResourceBlock where resourceBlockId in (1, 3);
```

##Assets

``` sql
-- sollte leer sein --------------------------------------
select * from AssetEntry where userId = 20538;
select * from AssetEntry where groupId = 16527;

-- change old mje user to new ----------------------------
update AssetEntry set userId = 10485 
  where userId = 20538;

delete from AssetEntry where groupId = 16527;
```

##DLFileEntry

``` sql
-- sollte leer sein --------------------------------------
select * from DLFileEntry where userId = 20538;

-- change old mje user to new ----------------------------
update DLFileEntry set userId = 10485 
  where userId = 20538;
```
  
##DLFolder

``` sql
-- sollte leer sein --------------------------------------
select * from DLFolder where userId = 20538;

-- change old mje user to new ----------------------------
update DLFolder set userId = 10485 
  where userId = 20538;
```

##MBMessage

``` sql
-- sollte leer sein --------------------------------------
select * from MBMessage where userId = 20538;
select * from MBMessage where groupId = 16527;

-- change old mje user to new ----------------------------
update MBMessage set userId = 10485 
  where userId = 20538;
delete from MBMessage where groupId = 16527;
```

##MBThread

``` sql
-- sollte leer sein --------------------------------------
select * from MBThread where lastPostByUserId = 20538;
select * from MBThread where rootMessageUserId = 20538;
select * from MBThread where groupId = 16527;

-- change old mje user to new ----------------------------
update MBThread set lastPostByUserId = 10485 
  where lastPostByUserId = 20538;
update MBThread set rootMessageUserId = 10485 
  where rootMessageUserId = 20538;
delete from MBThread where groupId = 16527;
```

##Set new Theme
posible themes are: politaktiv_WAR_politaktivtheme, politaktivdefault_WAR_politaktivdefaulttheme, classic

``` sql
update LayoutSet set themeId = 'politaktivdefault_WAR_politaktivdefaulttheme', wapThemeId = 'politaktivdefault_WAR_politaktivdefaulttheme'
update Layout set themeId = null, wapThemeId = null
```


#3. Start Server


#OneTime Stuff

6.0 Um AUI-Migration kümmern (ACHTUNG: Bereits durchgeführt, bei Dappsen auf github geforked) 
6.1 aptitude installieren, g++ installieren (beides benötigt für nodejs)
-> (sudo apt-get install g++ - aptitude is an alternative to apt-get)
-> sudo apt-get install build-essencial


6.1 nodejs installieren  (benötigt für upgrade tool)
Download von: http://nodejs.org/
cd [nodejsVerzeichnis]
./configure
make
sudo make install

6.2 upgrade tool installieren 
git clone https://github.com/PolitAktiv/liferay-aui-upgrade-tool.git
sudo npm install -g laut

6.2 upgrade tool anwenden
cd [liferaySdk/hooks]
laut -f politaktiv-theme/
(+ evtl empty theme)


6.0	Verändere im look-and-feel.xml die Versionsnummern von politaktiv-layouts, politaktiv-theme und politaktiv-empty-theme, sodass diese mit Liferay 6.2.0+ übereinstimmen
(ACHTUNG: auch schon auf Github)


9. Starte den Server. Im liferay-portal-6.2.0-ga1/deploy Verzeichnis befinden sich die WARs des layouts und der themes.
