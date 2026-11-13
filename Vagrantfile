
Vagrant.require_version ">= 2.2.8"

if Gem::Version.new(Vagrant::VERSION) < Gem::Version.new("2.4.0") &&
   !ENV["VAGRANT_EXPERIMENTAL"].to_s.match?(/\b(disks|1)\b/)
  abort <<~MSG
    [Vagrantfile] Vagrant #{Vagrant::VERSION} detectado. Defina VAGRANT_EXPERIMENTAL="disks".
  MSG
end

nome1      = "Guilherme Matias"
nome2      = "Bruno Anderson"
matricula1 = "20251380012"
matricula2 = "20251380014"
dominio    = "guilherme.bruno.devops"

[matricula1, matricula2].each do |matricula|
  unless matricula.match?(/\A\d{2,}\z/)
    abort("[Vagrantfile] Matrícula inválida.")
  end
end

mat1_fim = matricula1[-2, 2]
mat2_fim = matricula2[-2, 2]
xy       = matricula1[-1] + matricula2[-1]

rede     = "192.168.56"
ip_arq   = "#{rede}.1#{mat1_fim}"

oui_vbox = "080027"
mac_db   = "#{oui_vbox}#{xy}00#{mat1_fim}"
mac_app  = "#{oui_vbox}#{xy}00#{mat2_fim}"

ip_selecao_rede = "#{rede}.2"

fqdn = ->(vm) { "#{vm}.#{dominio}" }

descricao = lambda do |papel|
  "Projeto 01 DevOps - #{papel} - Equipe: #{nome1} e #{nome2}"
end

desabilitar_dhcp_virtualbox = lambda do |maquina|
  driver = maquina.provider.driver
  listagem = driver.execute_command(["list", "dhcpservers"])
  raise "VBoxManage falhou" unless listagem.exit_code.zero?

  listagem.stdout.to_s.split(/\r?\n[ \t]*\r?\n/).each do |bloco|
    campos = {}
    bloco.each_line do |linha|
      chave, valor = linha.split(":", 2)
      campos[chave.strip.downcase] = valor.strip unless valor.nil?
    end

    nome = campos["networkname"].to_s
    next if nome.empty?

    ips = campos.values_at("dhcpd ip", "ip", "loweripaddress", "upperipaddress").compact
    next unless ips.any? { |ip| ip.start_with?("#{rede}.") }
    next if campos["enabled"].to_s.casecmp?("no")

    [["--network", nome], ["--netname", nome]].any? do |opcao|
      driver.execute_command(["dhcpserver", "modify", *opcao, "--disable"]).exit_code.zero?
    end
  end
end

Vagrant.configure("2") do |config|

  config.vm.box              = "debian/bookworm64"
  config.vm.box_check_update = false
  config.ssh.insert_key      = false
  config.vm.synced_folder ".", "/vagrant", disabled: true

  config.vm.provider "virtualbox" do |vb|
    vb.memory               = 512
    vb.linked_clone         = true
    vb.check_guest_additions = false
  end

  config.trigger.before :up, :reload, :resume do |t|
    t.name     = "Desabilitar DHCP"
    t.on_error = :continue
    t.ruby do |_env, maquina|
      desabilitar_dhcp_virtualbox.call(maquina)
    end
  end

  config.vm.define "arq" do |arq|
    arq.vm.hostname = fqdn.call("arq")
    arq.vm.network "private_network", ip: ip_arq

    arq.vm.provider "virtualbox" do |vb|
      vb.name = fqdn.call("arq")
      vb.customize ["modifyvm", :id, "--description", descricao.call("arq")]
    end

    (1..3).each do |n|
      arq.vm.disk :disk, size: "10GB", name: "arq-disco#{n}"
    end
  end

  config.vm.define "db" do |db|
    db.vm.hostname = fqdn.call("db")
    db.vm.network "private_network", ip: ip_selecao_rede, mac: mac_db, auto_config: false

    db.vm.provider "virtualbox" do |vb|
      vb.name = fqdn.call("db")
      vb.customize ["modifyvm", :id, "--description", descricao.call("db")]
    end
  end

  config.vm.define "app" do |app|
    app.vm.hostname = fqdn.call("app")
    app.vm.network "private_network", ip: ip_selecao_rede, mac: mac_app, auto_config: false

    app.vm.provider "virtualbox" do |vb|
      vb.name = fqdn.call("app")
      vb.customize ["modifyvm", :id, "--description", descricao.call("app")]
    end
  end

  config.vm.define "cli" do |cli|
    cli.vm.hostname = fqdn.call("cli")
    cli.vm.network "private_network", ip: ip_selecao_rede, auto_config: false

    cli.vm.provider "virtualbox" do |vb|
      vb.name   = fqdn.call("cli")
      vb.memory = 1024
      vb.customize ["modifyvm", :id, "--description", descricao.call("cli")]
    end
  end
end
