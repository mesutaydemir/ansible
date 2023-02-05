```diff
- text in red
+ text in green
! text in orange
# text in gray
@@ text in purple (and bold)@@
```
# Ansible Notları
## Vagrant Kurulumu

## Vagrant Komutları
```
vagrant status
vagrant box list
vagrant up
vagrant destroy -f
vagrant --help
vagrant ssh ==> Varsayılan olarak kontrol makinesine bağlanır.
vagrant box --help
vagrant suspend
vagrant resume
vagrant init ubuntu/focal64 --box-version 20230119.0.0 ==> vagrant makinesini up etmeden önce oluşturulan vagrant dosyası
```
##Ansible kurulumu

### Öncelikle yazılım reposunu güncelleyip python3 virtual environment paketini [python3-venv] kurmuyoruz :) çünkü provisioning dosyasında var:
```
sudo apt update
sudo apt install python3-venv -y
```

Ev dizinimizin altında merkezi bir dosya altında yapılandırma dosyalarını oluşturuyoruz:
```
python3 -m venv ~/.venv/kamp
```
python3'ün path'ini belirtmek için aşağıdaki komutu çalıştırıyoruz:
```
source ~/.venv/kamp/bin/activate
```

Bu komutu etkisizleştirmek için: 
```
deactivate
```
`which python3` komutu ile */usr/bin/python3* görülecektir. `source ~/.venv/kamp/bin/activate` komutundan sonra *venv* altındaki python3 
```
pip3 install --upgrade pip # pip3 paketini güncelliyoruz.
```
### Dependancy management (pythonda kullanılacak kütüphaneleri indiriyoruz)
#### requirements.txt dosyasını /home/vagrant altında oluşturuyoruz:
```
nano requirements.txt
```
```
ansible==6.7.0
ansible-core==2.13.7
cffi==1.15.1
cryptography==39.0.0
Jinja2==3.1.2
MarkupSafe==2.1.2
packaging==23.0
pkg_resources==0.0.0
pycparser==2.21
PyYAML==6.0
resolvelib==0.8.1
```
### Aşağıdaki komutlar ile önce pip'i upgrade ediyoruz. Sonra requirements.txt içinde belirtilen python kütüphanelerini (versiyonlarında belirtildiği şekilde) güncelliyoruz. 

>Not: Kütüphanelerin versiyonları güvenlik güncellemeleri ve buglardan dolayı ara sıra güncellenerek kontrol edilmeli.
>pip3 freeze # requirements.txt dosyasındaki kütüphaneleri listelemek için

`pip3 install -r requirements.txt` (declarative)	==> `pip3 install ansible` yazarak da kurabilirdim (imperative)


# hosts dosyasını oluşturuyoruz. Yöneteceğimiz makineler bunlar demek:
```
nano hosts
```
```
host0 `#0d1117 cc` ansible_host=192.168.56.20
host1 ansible_host=192.168.56.21
host2 ansible_host=192.168.56.22

[all:vars]
ansible_user=vagrant
ansible_password=vagrant
```
## ad-hoc komutları ile önce bağlantıyı kontrol edeceğiz:

Kontrol makinesinde yönetilen makinelere bağlantı yapılabildiğini doğrulayalım:

ansible all -i hosts -m ping --ssh-common-args='-o StrictHostKeyChecking=no'

Alternatif olarak bu klasörde ansible.cfg dosyasını oluşturup aşağıdaki satırları ekledikten sonra:

[defaults]
host_key_checking = False
inventory=hosts

ansible all -i hosts -m ping 

komutunu çalıştırabiliriz.


# yanlış yazılan komutlarda rc ye bakıp 0 dışında bir değer ise failed dönüyor. Komut değişikliğe neden olan bir komut olmasa bile bizim ne yaptığımızı bilmediği için
CHANGED yazar.
(kamp) vagrant@control:~$ ansible all -i hosts -a "asds"
host2 | FAILED | rc=2 >>
[Errno 2] No such file or directory: b'asds'
host0 | FAILED | rc=2 >>
[Errno 2] No such file or directory: b'asds'
host1 | FAILED | rc=2 >>
[Errno 2] No such file or directory: b'asds'

ansible host0 -i hosts -m service -a "name=sshd state=started"
	host0 | SUCCESS => {


## Bazı tek seferlik işlemler için playbook yazıp bunu çalıştırmak anlamsız olabilir. örneğin makinelerin kullandıkları ram miktarını öğrenebiliriz:

ansible all -a "free -m"

ya da disk boyutu

ansible all -a "df -h /"

## host00 ve host01'in kullandığı ram miktarını kontrol etmek için
ansible 'all:host0,host1' -a "df -h /"
ansible 'all:!host2' -i hosts -a "ls" #host2 dışındaki tüm makineler. Sunucu kategorisi de kullanılmışsa [databsase] all yerine bu kategori isimleri ile de işlem yapılabilir.

## makine ne kadar ayakta
ansible host0 -a "uptime"

## -m parametresi ile modül çağırıp ad-hoc komutlarda bunları kullanabiliriz. Bir işlemin modülü varsa shell script yerine modül ile işlem gerçekleştirilme tercih edilmeli.
ansible-doc [modül adı]
ansible-doc service
ansible host0 -m shell -a "systemctl status sshd | grep running"
ansible all -m shell -a "systemctl status sshd | grep running"
ansible host0 -m shell -a "lsb_release -a"
ansible all -m shell -b -a "reboot" [Tüm makineri reboot et. -b: become sudo sudo kullanmasak iyi olur çünkü parola soracaktır. onun yerine -b ile become sudo]. Gerekmeyen hiçbir parametre kullanılmamalı. Örneğin bazı komutlar sudo yetkisi gerektirmiyorsa -b ye egrek yok.
ansible all -m service -b  -a "name=sshd state=restarted" # service kullanılarak yapılan bir cli işlemi
ansible all -m shell -b -a "shutdown -h now"
ansible host0 -m setup | less
ansible all -m copy -a "src=/etc/hosts dest=/tmp/hosts mode=600 owner=ubuntu group=ubuntu" #kullanıcı adı ubuntu ve grubu ubuntu olan kullanıcı ve grub için 600 erişim hakkı verilmesi için


## Idempotency
bir kere çalıştırılıp elde edilen çıktı sonrasında aynı komut çalıştırıldığında alınan hata. örneğin mkdir.

## CLI'den help dokumanını görüntülemek için:
ansible-doc [paket_adı]
ansible-doc apt


## apt ile host2'ye ansible'da shell dışında bir modül kullanarak apache2'yi kuralım.
ansible host2 -b -m apt -a "name=apache2 state=present update_cache=yes" # varolan en güncel sürüm
ansible host2 -b -m service -a "name=apache2 state=started" # servisin çalışıp çalışmadığını kontrol etmek ve durmuşsa başlatacak. 
ansible host2 -b -m apt -a "name=apache2 state=latest update_cache=yes" # yeni versiyon geldiğinde present var mı yok muyu kontrol eder. latest'ta ise güncel sürüm var mı kontrole der ve kurar.
ansible host2 -b -m apt -a "name=apache2=2.4.41-4ubuntu3.13 state=latest update_cache=yes" 
## Versiyon downgrade etmek istiyorsak hata verecektir. downgrade için parametre vermek gerekiyor. Bunu araştır!
apt-cache madison apache2 # apache2 paketinin repodaki mevcut sürümlerini gösterir.
ansible host2 -b -m apt -a "name=apache2=2.4.41-4ubuntu3.13 state=latest update_cache=yes" 

# Paket kaldırmak için:
[remove] ansible host2 -b -m apt -a "name=apache2=2.4.41-4ubuntu3.13 state=absent"
[purge] ansible host2 -b -m apt -a "name=apache2=2.4.41-4ubuntu3.13 state=absent purge=yes" # "yes" alternatives: "true";"on"

## Nginx kurulumu
ansible host2 -b -m apt -a "name=nginx state=latest update_cache=yes"
ansible host2 -b -m service -a "name=nginx state=started" # servisin çalışıp çalışmadığını kontrol etmek ve durmuşsa başlatacak. 

## downgrade için -allow_downgrade seçeneği kullanılacak. Bununla ilgili localde testler yapılabilir.

================================================================================================================
# INVENTORY
## /home/mesut/Desktop/ansible/inventory-main/hosts dosyasındaki örnek dosyayı incele!
- Geçerli bir fqdn verilirse ansible_host tanımlaması yapılmaya gerek kalmayabilir.
- ansible_port belirtilerek default portlar dışında da portlar kullanılabilir: (örneğin ansible_port=5555)
- parent children ilişkisi ile aynı türden fakat farklı kategorideki sunucular için ortak işlemler gerçekleştirilebilir:
	# webservers in all geos
	[webservers:children]
	atlanta_webservers
	boston_webservers

## inventory dosyası yaml formatında da olabilir:
all:
  hosts:
    mail.example.com:
  children:
    webservers:
      hosts:
        foo.example.com:
        bar.example.com:
    dbservers:
      hosts:
        one.example.com:
        two.example.com:
        three.example.com:
    east:
      hosts:
        foo.example.com:
        one.example.com:
        two.example.com:
    west:
      hosts:
        bar.example.com:
        three.example.com:
    prod:
      hosts:
        foo.example.com:
        one.example.com:
        two.example.com:
    test:
      hosts:
        bar.example.com:
        three.example.com:


===============================================================================================================
YAML [https://docs.ansible.com/ansible/latest/reference_appendices/YAMLSyntax.html]
.yaml dokumanları 3 tire (---) ile başlar 3 nokta ile biter.

===============================================================================================================
# PLAYBOOK
playbook'u değiştirmeden yalnızca host2'de işlem yapmak istiyorsak:
ansible-playbook nginx.yml --limit "host0,host1"

git restore nginx.yml komutu ile git'den clonelanan bir dosyayı ile haline getiriyoruz.

# gather_facts'i playbook çalıştırıldığında disable etmek için tags -configuration gather_facts: no

---
- name: Remove nginx
  hosts: all
  become: True
  tags:
    - configuration
  gather_facts: no
  tasks:
  - name: Remove nginx package
    apt:
      name: nginx
      state: absent

ssh-keygen -R 192.168.56.20 komutu ile hedef makinedeki ssh-keygen yeniden yapılandırılıyor.
