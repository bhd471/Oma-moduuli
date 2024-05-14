# Oma miniprojekti

Moduulin tarkoituksena on nopeuttaa palveluiden asentamista. Moduulia voidaan soveltaa toimimaan suurella määrällä koneita.

Projektissa luodaan Vagrantilla kaksi virtuaalikonetta, joille luodaan master-slave arkkitehtuuri. Asennetaan Apache, Ufw, Curl, Git ja Micro. 

Lisenssi: GNU General Public License v.3

Tekijä: Janika Penttinen

## Master-slave koneiden määrittäminen

Aloitin luomalla kaksi uutta virtuaalikonetta Vagrantin avulla.

	$ vagrant init debian/bullseye64
 	$ notepad VagrantFile
  
Määritin koneet tekemällä muutoksia Vagrantfileen.


    # -*- mode: ruby -*-
    # vi: set ft=ruby :

    $tscript = <<TSCRIPT
    set -o verbose
    apt-get update
    apt-get -y install tree
    TSCRIPT

    Vagrant.configure("2") do |config|
	config.vm.synced_folder ".", "/vagrant", disabled: true
	config.vm.synced_folder "shared/", "/home/vagrant/shared", create: true
	config.vm.provision "shell", inline: $tscript
	config.vm.box = "debian/bullseye64"

	    config.vm.define "kone1" do |kone1|
		kone1.vm.hostname = "kone1"
		kone1.vm.network "private_network", ip: "192.168.88.101"
	 end

	    config.vm.define "kone2", primary: true do |kone2|
		kone2.vm.hostname = "kone2"
		kone2.vm.network "private_network", ip: "192.168.88.102"
    	end
	
    end
Sitten koneet käyntiin ja kirjauduttiin sisään. Kokeilin kirjautua molemmille koneille ssh avulla.

        $ vagrant up
        $ vagrant ssh kone1
        $ vagrant exit
        $ vagrant ssh kone2
        
Pingasin koneilla toisiaan, toimii!

        $ vagrant ssh kone1
        $ ping -c 192.168.88.102
        $ exit

        $ vagrant ssh kone2
        $ ping -c 1 192.168.88.101
        $ exit

![image](https://github.com/bhd471/Palvelinten-hallinta/assets/148760837/06365af0-5754-4c5c-a3ff-08a8467460a1)

Asetetaan "kone1" masteriksi ja tarkistetaan sen IP-osoite.

    $ sudo apt-get update
    $ sudo apt-get -y install salt-master
    $ hostname -I

Asetetaan "kone2" orjaksi.

    $ sudo apt-get update
    $ sudo apt-get -y install salt-minion

Liitin koneet yhteen lisäämällä masterin IP-osoite konfiguraatiotiedostoon ja käynnistin orjan uudelleen.

        $ sudoedit /etc/salt/minion
        $ sudo systemctl restart salt-minion.service
    

Hyväksyin orja-avaimen master koneella.

        $ sudo salt-key -A

![image](https://github.com/bhd471/Palvelinten-hallinta/assets/148760837/89657170-6045-45f8-826f-7f8621574d4d)

## Apache & testisivun korvaaminen

Asensin Apachen ja curlin ensin manuaalisesti.

    $ sudo apt-get -y install apache2
    $ sudo apt-get -y install curl

Loin /srv/salt -hakemiston, johon loin apache.sls tiedoston. Loin tilan, joka asentaa ja käynnistää Apachen, ja korvaa testisivun.

        apache2:
          pkg.installed

        apache2 Service:
          service.running:
            - name: apache2
            - enable: True
            - require:
              - pkg: apache2
    
        /var/www/html/index.html:
          file.managed:
          - source: salt://apache2/index.html
    

## Tulimuuri

Lähdin asentamaan tulimuuria. Ensin manuaalisesti, sitten automatisoin asennuksen.

        $ sudo apt-get update
        $ sudo apt-get -y install ufw

Loin uuden hakemiston /srv/salt/ufw, johon loin ufw.sls -tiedoston. 
Tässä kohtasin ongelmia. Onnistuin luomaan tilan, joka asentaa ja käynnistää tulimuurin. Yritin tämän lisäksi tehdä reikää tulimuuriin. En kuitenkaan onnistunut tässä.

        ufw:
          pkg.installed

        ufw Service:
          service.running:
            - name: ufw
            - enable: True
            - require:
              - pkg: ufw

        ufw Allow:
          ufw.allow:
            - port: 22
            - proto: tcp

![image](https://github.com/bhd471/Palvelinten-hallinta/assets/148760837/a601a386-f7f5-4c00-9081-e78310013f62)

Tässä vaiheessa luovutin portina avaamisen automatisoinnin kanssa ja avasin portin 22 manuaalisesti, jotta pystyn jatkamaan SSH-kirjautumista

        $ sudo ufw allow 22/tcp

## Curl

Automatisoin curlin asennuksen.
Loin kansion 'curl', johon loin curl.sls -tiedoston. Lisäsin tilan tiedostoon. Lisäsin top.sls -tiedostoon curlin.

        curl:
          pkg.installed



## Git

Lähdin asentamaan Gittiä. Ensin manuaalisesti, sitten automatisoin asennuksen. 

        $ sudo apt-get update
        $ sudo apt-get -y install git
        

Loin uuden hakemiston /srv/salt/git, johon loin uuden git.sls -tiedoston. 

        git:
          pkg.installed

Lisäsin Gitin top.sls -tiedostoon. 


## Micro

Lähdin asentamaan Micro-editoria. Ensin manuaalisesti, sitten automatisoin asennuksen. Loin uuden hakemiston /srv/salt/micro, johon loin uuden micro.sls -tiedoston. 

        micro:
          pkg.installed
Lisäsin micron top.sls -tiedostoon. 
        
## Lopputulos

Top.sls -tiedoston sisältö:

![image](https://github.com/bhd471/Palvelinten-hallinta/assets/148760837/03d0890e-c797-4de4-b2ed-d8c5e8f81275)

Kaikki tilat ajettiin onnistuneesti:

        $ sudo salt 'kone2' state.apply

![image](https://github.com/bhd471/Palvelinten-hallinta/assets/148760837/07658458-dc97-4375-83fe-2bb2aa325e03)


![image](https://github.com/bhd471/Palvelinten-hallinta/assets/148760837/44c315cf-2265-4329-8d47-6e390b8a3d32)

Tältä näyttää localhost-sivun sisältö:

![image](https://github.com/bhd471/Palvelinten-hallinta/assets/148760837/55fd73cf-f82e-44c2-91bb-ce30ce62131e)

### Lähteet

Karvinen, T. 21.03.2023. H7 - Oma miniprojekti. Palvelinten hallinta -kurssi, kevät 2024. Tero Karvisen verkkosivut. Luettavissa: https://terokarvinen.com/2024/configuration-management-2024-spring/#comments. Luettu: 11.05.2024.

Linode. 19.10.2018. Configure Apache with Salt Stack. Luettavissa: https://www.linode.com/docs/guides/configure-apache-with-salt-stack/. Luettu: 13.05.2024.

VMware. 30.04.2024. Salt states firewall. Salt project. Luettavissa: https://docs.saltproject.io/en/latest/ref/states/all/salt.states.firewall.html. Luettu: 13.05.2024.
