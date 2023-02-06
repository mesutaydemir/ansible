# Ansible Notları
## Vagrant Kurulumu
Aşağıdaki komutlar ile sırasıyla *vagrant*ı Ubuntu makinemize kurup versiyonunu kontrol ediyoruz.
```
sudo apt install vagrant
vagrant --v
```
## Vagrant Komutları
```
vagrant status ==> tüm makinelerin durumunu kontrol etmek için kullanılır.
vagrant box list
vagrant up ==> Vagrant dosyasındaki verilere göre makineleri oluşturur.
vagrant destroy -f ==> tüm makineleri silmek için kullanılır.
vagrant --help
vagrant ssh ==> Varsayılan olarak kontrol makinesine bağlanır.
vagrant box --help
vagrant suspend ==> tüm makineleri uyku moduna alır.
vagrant resume ==> tüm makineleri uyku modundan çıkarıp başlatır.
vagrant init ubuntu/focal64 --box-version 20230119.0.0 ==> vagrant makinesini up etmeden önce oluşturulan vagrant dosyası
```
### Örnek Bir Vagrant Dosyası Aşağıdaki gibidir:
```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64"
  config.vm.box_version = "20230119.0.0"

  config.vm.define "control", primary: true do |control|
    control.vm.hostname = "control"

    control.vm.network "private_network", ip: "192.168.56.10"

    control.vm.provision "shell", inline: <<-SHELL
      apt update > /dev/null 2>&1 && apt install -y python3.8-venv sshpass > /dev/null 2>&1
    SHELL
  end

  (0..2).each do |i|
    config.vm.define "host#{i}" do |node|
      node.vm.hostname = "host#{i}"

      node.vm.network "private_network", ip: "192.168.56.#{i+20}"

      node.vm.provision "shell", inline: <<-SHELL
        sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config    
        systemctl restart sshd.service
      SHELL
    end
  end
end
```

## Ansible kurulumu

- Öncelikle yazılım reposunu güncelleyip python3 virtual environment paketini *python3-venv* kurmayabiliriz :) çünkü yukarıdaki Vagrant Dosyasında (provisioning file) var:

```
sudo apt update
sudo apt install python3-venv -y
```

- Ev dizinimizin altında merkezi bir dosya altında yapılandırma dosyalarını oluşturuyoruz:
```
python3 -m venv ~/.venv/kamp
```
python3'ün path'ini belirtmek için aşağıdaki komutu çalıştırıyoruz:
```
source ~/.venv/kamp/bin/activate
```

- Bu komutu etkisizleştirmek için: 
```
deactivate
```
`which python3` komutu ile */usr/bin/python3* görülecektir. `source ~/.venv/kamp/bin/activate` komutundan sonra *venv* altındaki python3 
```
pip3 install --upgrade pip # pip3 paketini güncelliyoruz.
```
### Dependancy management
- pythonda kullanılacak kütüphaneleri tanımlamak için *requirements.txt* dosyasını /home/vagrant altında oluşturuyoruz:
```
nano requirements.txt
```
- Dosyanın içine aşağıdaki satrıları ekledikten sonra kaydedip çıkıyoruz:
```ruby
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
- Aşağıdaki komutlar ile önce pip'i upgrade ediyoruz. Sonra *requirements.txt* içinde belirtilen python kütüphanelerini (versiyonlarında belirtildiği şekilde) güncelliyoruz. 

>Not: Kütüphanelerin versiyonları güvenlik güncellemeleri ve buglardan dolayı ara sıra güncellenerek kontrol edilmeli.

`pip3 freeze` requirements.txt dosyasındaki kütüphaneleri listelemek için

`pip3 install -r requirements.txt` **declarative**
`pip3 install ansible` yazarak da kurabilirdim **imperative**

- hosts dosyasını oluşturuyoruz. Yöneteceğimiz makineleri bu dosyada tanımlıyoruz.
```
nano hosts
```
- Oluşturduğumuz host dosyasının içeriği aşağıdaki gibi:
```ruby
host0 ansible_host=192.168.56.20
host1 ansible_host=192.168.56.21
host2 ansible_host=192.168.56.22

[all:vars]
ansible_user=vagrant
ansible_password=vagrant
```
## Ad-hoc 
Bu komutlar ile önce bağlantıyı kontrol edeceğiz:

- Kontrol makinesinde yönetilen makinelere bağlantı yapılabildiğini doğrulayalım:

```
ansible all -i hosts -m ping --ssh-common-args='-o StrictHostKeyChecking=no'
```

- Alternatif olarak bu klasörde *ansible.cfg* dosyasını oluşturup aşağıdaki satırları ekledikten sonra:
```
[defaults]
host_key_checking = False
inventory=hosts
```
aşağıdaki komutu çalıştırabiliriz:
```
ansible all -i hosts -m ping 
```

- Yanlış yazılan komutlarda rc değerinde 0 dışında bir değer dönüyorsa *failed* dönüyor. Komut değişikliğe neden olan bir komut olmasa bile bizim ne yaptığımızı bilmediği için *CHANGED* yazacaktır.
```
(kamp) vagrant@control:~$ ansible all -i hosts -a "asds"
```
```ruby
host2 | FAILED | rc=2 >>
[Errno 2] No such file or directory: b'asds'
host0 | FAILED | rc=2 >>
[Errno 2] No such file or directory: b'asds'
host1 | FAILED | rc=2 >>
[Errno 2] No such file or directory: b'asds'
```

- Servis modülü ile sshd servisinin çalışıp çalışmadığını görmek için aşaıdaki komut kullanılabilir:
```
ansible host0 -i hosts -m service -a "name=sshd state=started"
```
- Komutun çıktısı aşağıdaki gibi olacaktır:
```
host0 | SUCCESS => {
```

- Bazı tek seferlik işlemler için playbook yazıp bunu çalıştırmak anlamsız olabilir. Örneğin makinelerin kullandıkları ram miktarını öğrenebiliriz:
```
ansible all -a "free -m"
```
ya da disk boyutu
```
ansible all -a "df -h /"
```
- host00 ve host01'in kullandığı ram miktarını kontrol etmek için
```
ansible 'all:host0,host1' -a "df -h /"
```
- host2 dışındaki tüm makinelerde *ls* komutunu çalıştırmak için aşağıdaki komut kullanılabilir. Sunucu kategorisi de kullanılmışsa [databsase] all yerine bu kategori isimleri ile de işlem yapılabilir.
```
ansible 'all:!host2' -i hosts -a "ls" #
```

- makine ne kadar zamandır çalışır durumda olduğunu görmek için aşağıdaki komut kullanılabilir:
```
ansible host0 -a "uptime"
```

- -m parametresi ile modül çağırıp ad-hoc komutlarda bunları kullanabiliriz. Bir işlemin modülü varsa shell script yerine modül ile işlem gerçekleştirilme tercih edilmeli. Kullanılmak istenen modülün dokumantasyonuna aşağıdaki komutla ulaşılabilir:
```
ansible-doc [modül adı]
ansible-doc service
```
- Aşağıda bazı modüllerin kullanım örnekleri verilmektedir:
```
ansible host0 -m shell -a "systemctl status sshd | grep running"
ansible all -m shell -a "systemctl status sshd | grep running"
ansible host0 -m shell -a "lsb_release -a"
```
- *sudo* komutunu doğrudan ansible komutunda kullanmasak iyi olur çünkü parola sorabilir ve otomasyonda parola girme ya da onaylama adımı yok. Bunun yerine *-b* ile *become sudo*]. Gerekmeyen hiçbir parametre kullanılmamalı. Örneğin bazı komutlar sudo yetkisi gerektirmiyorsa *-b* ye gerek yok.
```
ansible all -m shell -b -a "reboot"
ansible all -m service -b  -a "name=sshd state=restarted"
ansible all -m shell -b -a "shutdown -h now"
ansible host0 -m setup | less
```
- Kullanıcı adı ubuntu ve grubu ubuntu olan kullanıcı ve grub için 600 erişim hakkı verilmesi için
```
ansible all -m copy -a "src=/etc/hosts dest=/tmp/hosts mode=600 owner=ubuntu group=ubuntu" 
```

- apt ile host2'ye ansible'da shell dışında bir modül kullanarak apache2'yi kuralım. Varolan en güncel sürümü kurmak için *state=present* ya da *state=latest* kullanılabilir:
```
ansible host2 -b -m apt -a "name=apache2 state=present update_cache=yes" 
```
- Servisin çalışıp çalışmadığını kontrol etmek ve durmuşsa başlatmak için *state=started* komutu kullanılabilir:
```
ansible host2 -b -m service -a "name=apache2 state=started"  
```
- Yeni versiyon geldiğinde *present* var mı yok muyu kontrol eder. *latest* ise güncel sürüm var mı kontrol eder ve kurar:
```
ansible host2 -b -m apt -a "name=apache2 state=latest update_cache=yes" 
ansible host2 -b -m apt -a "name=apache2=2.4.41-4ubuntu3.13 state=latest update_cache=yes" 
```
- Versiyon downgrade etmek istiyorsak hata verecektir. Downgrade için -allow_downgrade seçeneği kullanılacak. Bununla ilgili localde testler yapılabilir.
```
ansible host2 -b -m apt -a "name=apache2=2.4.41-4ubuntu3.13 state=latest update_cache=yes" 
```
- apache2 paketinin repodaki mevcut sürümlerini görmek için aşağıdaki komut kullanılabilir:
```
apt-cache madison apache2
```
- Paket kaldırmak için:
```
ansible host2 -b -m apt -a "name=apache2=2.4.41-4ubuntu3.13 state=absent"
ansible host2 -b -m apt -a "name=apache2=2.4.41-4ubuntu3.13 state=absent purge=yes"
```
- Nginx kurulumu için aşağıdaki komut kullanılabilir:
```
ansible host2 -b -m apt -a "name=nginx state=latest update_cache=yes"
ansible host2 -b -m service -a "name=nginx state=started" # servisin çalışıp çalışmadığını kontrol etmek ve durmuşsa başlatacak. 
```

## Inventory
- /home/mesut/Desktop/ansible/inventory-main/hosts dosyasındaki örnek dosyayı incele!
- Geçerli bir fqdn verilirse ansible_host tanımlaması yapılmaya gerek kalmayabilir.
- ansible_port belirtilerek default portlar dışında da portlar kullanılabilir: (örneğin ansible_port=5555)
- parent children ilişkisi ile aynı türden fakat farklı kategorideki sunucular için ortak işlemler gerçekleştirilebilir:
```ruby
#webservers in all geos
[webservers:children]
atlanta_webservers
boston_webservers
```

- Inventory dosyası yaml formatında da olabilir:
```ruby
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
```
- Bir başka inventory dosyası örneği ise aşağıdaki gibidir. Burada server grupları, ansible'ın erişilebileceği port (varsayılan 22/TCP), ssl_private_key_file konumu, become kullanıcısı ve paroları gibi parametrelerin kullanım örnekleri görlüebilir.  

```ruby
[atlanta_webservers]
www-atl-1.example.com ansible_user=myuser ansible_port=5555 ansible_host=192.0.2.50
www-atl-2.example.com ansible_user=myuser ansible_host=192.0.2.51

[boston_webservers]
www-bos-1.example.com ansible_user=myotheruser ansible_ssh_private_key_file=/path/to/your/.ssh/file_name
www-bos-2.example.com ansible_user=myotheruser ansible_ssh_private_key_file=/path/to/your/.ssh/file_name

[atlanta_dbservers]
db-atl-[1:4].example.com ansible_user=myotheruser ansible_become_user=mybecomeuser ansible_become_password=mybecomeuserpassword

[boston_dbservers]
db-bos-1.example.com
# ip is not a good usage but is valid
192.0.2.59

# webservers in all geos
[webservers:children]
atlanta_webservers
boston_webservers

# dbservers in all geos
[dbservers:children]
atlanta_dbservers
boston_dbservers

# everything in the atlanta geo
[atlanta:children]
atlanta_webservers
atlanta_dbservers

# everything in the boston geo
[boston:children]
boston_webservers
boston_dbservers

[altanta:vars]
proxy=proxy.atlanta.example.com

[all:vars]
host_key_checking = false

```
> YAML Syntax ile ilgili dokumantasyona [YAML Syntax](https://docs.ansible.com/ansible/latest/reference_appendices/YAMLSyntax.html) adresinden erişilebilir.

## Playbook
- Playbook playlerden oluşur. En tepede liste olarak playler vardır. Playler üç tire (*---*) ile başlar ve 3 tire (*---*) ile bir sonraki play'e geçilir.
- Playbook'u değiştirmeden yalnızca **host2**' de işlem yapmak istiyorsak:
```
ansible-playbook nginx.yml --limit "host0,host1"
ansible-playbook nginx.yml --limit "!host2"
```

>`git restore nginx.yml` komutu ile git'den clonelanan bir dosyayı ile haline getiriyoruz.

- Nginx'i kurmak için kullanılacak *nginx.yml* playbook dosyasının içi aşağıdaki gibidir:
```ruby
---
- name: Install nginx
  hosts: all
  become: True
  gather_facts: no ==> *ansible-playbook abc.yml* komutu çalıştırıldığında gact gathering devre dışı bırakılır.
  tasks:
  - name: Install nginx package
    apt:
      name: nginx
      update_cache: True ==> sudo apt update komutunun yaptığı işlem olan yazılım repolarının listesini güncelliyor.
```
> gather_facts'i playbook çalıştırıldığında disable etmek için `gather_facts: no` değeri kullanılır.

- Nginx'i kaldırmak için kullanılacak nginx.yml playbook dosyasının içi aşağıdaki gibidir:
```ruby
---
- name: Remove nginx
  hosts: all
  become: True
  gather_facts: no
  tasks:
  - name: Remove nginx package
    apt:
      name: nginx
      state: absent
```
>`ssh-keygen -R 192.168.56.20` komutu ile hedef makinedeki ssh-keygen yeniden yapılandırılıyor.

- Şimdi `updat_cache=True` ile yazılım repolarının her seferinde güncellenmesi yerine örneğin 1 gün güncellenmemişse yazılım reposunu güncelleme işlemini gerçekleştirelim [**cache_valid_time: 600**]

```ruby
- name: Install nginx
  hosts: all
  become: True
  gather_facts: no
  tasks:
  - name: Install nginx package
    apt:
      name: nginx
      update_cache: True
      cache_valid_time: 6000
```
- Çalışan task'ın ne kadar sürdüğü ve çalışıp çalışmadığını görmek için *ansible.cfg* dosyası içinde `callback_whitelist = profile_tasks` ayarlamamız gerekiyor.
```ruby
[defaults]
host_key_checking = False
inventory=hosts
callbacks_enabled = profile_tasks
```
- Toplamda tüm taskların ne kadar sürede tamamlandığını görmek için `callbacks_enabled = profile_tasks` kullanılır.
```ruby
[defaults]
host_key_checking = False
inventory=hosts
callbacks_enabled = timer
```
- `register: nginx_result` kullanılarak playbook çalıştırıldığında CLI çıktısında bir değişiklik göstermiyor. `debug` modülüyle nginx_result'ı göstereceğiz.
```ruby
- name: Install nginx
  hosts: all
  become: True
  gather_facts: no
  tasks:
  - name: Install nginx package
    apt:
      name: nginx
      update_cache: True
      cache_valid_time: 6000
    register: nginx_result
```
- `debug` modülüyle nginx_result'ı göstermek için `name: Print result` task altındaki parametreler kullanılacak:
```ruby
- name: Install nginx
  hosts: all
  become: True
  gather_facts: no
  tasks:
  - name: Install nginx package
    apt:
      name: nginx
      update_cache: True
      cache_valid_time: 6000
    register: nginx_result
  - name: Print result
    debug:
      var: nginx_result
```
Yukarıdaki playbook çalıştırıldığnda komut satırında aşağıdaki gibi bir çıktı görülecektir:
```ruby
TASK [Print result] ******************************************************************************************************************************************************
ok: [host0] => {
    "nginx_result": {
        "cache_update_time": 1675671772,
        "cache_updated": true,
        "changed": false,
        "failed": false
    }
}
ok: [host1] => {
    "nginx_result": {
        "cache_update_time": 1675671782,
        "cache_updated": true,
        "changed": false,
        "failed": false
    }
}
ok: [host2] => {
    "nginx_result": {
        "cache_update_time": 1675671808,
        "cache_updated": true,
        "changed": false,
        "failed": false
    }
}

```
- `var: nginx_result` yerine `var: nginx_result.cache_updated` yazarak yalnızca *cache_updated* bilgisi alınabilir:
```ruby
- name: Install nginx
  hosts: all
  become: True
  gather_facts: no
  tasks:
  - name: Install nginx package
    apt:
      name: nginx
      update_cache: True
      cache_valid_time: 6000
    register: nginx_result
  - name: Print result
    debug:
      var: nginx_result.cache_updated
```
-  çıktı json yerine yaml formatında görmek için *ansible.cfg dosyasına `stdout_callback = yaml` parametresi eklenebilir:
```ruby
[defaults]
host_key_checking = False
inventory=hosts
callbacks_enabled = profile_tasks
stdout_callback = yaml
```
- `copy` modülü ile özellikle şablon görevi gören dosyalar *src* ve *dest* ile kaynak ve hedef konumları belirtilerek dosyalar kopyalanır. Ayrıca tasklarda `notify` ile belirtilen ilgili handler altındaki tasklar yapılır. Aşağıdaki örnekte jjna2 {{nginx_conf}} şablon dosyası host makinelerde /etc/nginx/conf.d altına example.com.conf olarak sahibi ve sahibinin grubu root olacak şekilde 644 erişim izni ile kopyalanıyor. Daha sonra notify ile handlers altında belirtilen nginx'i yeniden başlatma task'ı çağrılıyor:

```ruby
  ---
- name: Install and configure nginx
  hosts: all
  become: True
  vars:
    nginx_conf: >
      server {
          listen       80;
          server_name  example.com;

          location / {
              proxy_pass      http://127.0.0.1:8000;
          }
      }

  tasks:
  - name: Install nginx package
    apt:
      name: nginx
      update_cache: True
      cache_valid_time: 60000

  - name: Update nginx configuration
    copy:
      content: "{{ nginx_conf }}"
      dest: /etc/nginx/conf.d/example.com.conf
      owner: root
      group: root
      mode: '0644'
    notify: Reload nginx
  
   - debug: msg=After copy
   
  handlers:
  - name: Reload nginx
    service:
      name: nginx
      state: reloaded

```
- `server_name`e *sub.example.com* eklendikten sonra `debug: msg="After copy"` taskı eklenip playbook yeniden çalıştırıldığında
```ruby
---
- name: Install and configure nginx
  hosts: all
  become: True
  vars:
    nginx_conf: >
      server {
          listen       80;
          server_name  example.com sub.example.com;

          location / {
              proxy_pass      http://127.0.0.1:8000;
          }
      }

  tasks:
  - name: Install nginx package
    apt:
      name: nginx
      update_cache: True
      cache_valid_time: 60000

  - name: Update nginx configuration
    copy:
      content: "{{ nginx_conf }}"
      dest: /etc/nginx/conf.d/example.com.conf
      owner: root
      group: root
      mode: '0644'
    notify: Reload nginx
  
  - debug: msg="After copy"
   
  handlers:
  - name: Reload nginx
    service:
      name: nginx
      state: reloaded
```
- handler'ın hemen çalışmasını istediğimiz durumlar olabilir. db kuruldu. config değiştirdi. restart edip user create etmek istediğimizde  metatask olarak `ansible.builtin.meta: flush_handlers` kullanılacilir. Aşağıdaki örnekte debug'tan önce `metatask` kullanılmıştır:

```ruby
---
- name: Install and configure nginx
  hosts: all
  become: True
  vars:
    nginx_conf: >
      server {
          listen       80;
          server_name  example.com sub.example.com;

          location / {
              proxy_pass      http://127.0.0.1:8000;
          }
      }

  tasks:
  - name: Install nginx package
    apt:
      name: nginx
      update_cache: True
      cache_valid_time: 60000

  - name: Update nginx configuration
    copy:
      content: "{{ nginx_conf }}"
      dest: /etc/nginx/conf.d/example.com.conf
      owner: root
      group: root
      mode: '0644'
    notify: Reload nginx
  
  - name: Force all notified handlers
    ansible.builtin.meta: flush_handlers
  
  - debug: msg="After copy"
   
  handlers:
  - name: Reload nginx
    service:
      name: nginx
      
```
- InventoryWdeki değerlere erişebiliyoruz. Her bir host için kendisi için tanımlanan fact'ler basılacak *hostvars["host1"].myvar]* Aşağıdaki örnekteki hosts dosyasındaki `myvar` değerlerine bakalım:
```ruby
host0 ansible_host=192.168.56.20 myvar=host0val
host1 ansible_host=192.168.56.21 myvar=host1val
host2 ansible_host=192.168.56.22 myvar=host2val

[all:vars]
ansible_user=vagrant
ansible_password=vagrant
```
Aşağıdaki *nginx.yml* playbook'ta *task*ta tüm host'lar için *host1val* fact'i görülecektir:
```ruby
---
- name: Install nginx
  hosts: all
  become: True
  tasks:
  - name: Install nginx package
    apt:
      name: nginx
      update_cache: True
      cache_valid_time: 60000
    register: nginx_result

  - name: Print result
    debug:
      var: nginx_result.cache_updated
  - name: Print result
   debug:
     var: hostvars["host1"]["myvar"]
```
Yukarıdaki *nginx.yml* playbook'un çıktısı aşağıdaki gibi olacaktır:
```ruby
ok: [host0] => 
  hostvars["host1"].myvar: host1val
ok: [host1] => 
  hostvars["host1"].myvar: host1val
ok: [host2] => 
  hostvars["host1"].myvar: host1val
```
Şimdi başka bir örneğe bakalım. Host dosyamız aşağıdaki gibi olsun:
```ruby
host0 ansible_host=192.168.56.20 myvar=host0val
host1 ansible_host=192.168.56.21 myvar=host1val
host2 ansible_host=192.168.56.22 myvar=host2val another-val=x

[all:vars]
ansible_user=vagrant
ansible_password=vagrant
```
Host2'deki *another-val=x* değerini tüm tasklarda görmek için nginx.yml playbook'u aşağıdaki gibi olacaktır:
```ruby                                                                          
---
- name: Install nginx
  hosts: all
  become: True
  tasks:
  - name: Install nginx package
    apt:
      name: nginx
      update_cache: True
      cache_valid_time: 60000
    register: nginx_result

  - name: Print result
    debug:
      var: hostvars["host1"].myvar

  - name: Print result
    debug:
      var: hostvars["host2"]["another-val"]

```
- Playbook içinde register dışında (farklı bir task çalıştırmayacağız) fact (variable) set etmek. Aşağıdaki örnekte `is_cache_updated` :
```ruby
---
- name: Install nginx
  hosts: all
  become: True
  tasks:
  - name: Install nginx package
    apt:
      name: nginx
      update_cache: True
      cache_valid_time: 60000
    register: nginx_result

  - name: Print result
    debug:
      var: is_cache_updated
```

## Modules

## Variables and Facts

## Conditionals

## Loops

## Roles

## Additional topics(Ansible Galaxy, AWX)
