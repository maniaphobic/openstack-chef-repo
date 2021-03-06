current_dir = File.dirname(__FILE__)
client_opts = "--force-formatter --no-color -z --config #{current_dir}/.chef/knife.rb"

task default: ["test"]

desc "Default gate tests to run"
task :test => [:rubocop, :berks_vendor, :json_check]

def run_command(command)
  if File.exist?('Gemfile.lock')
    sh %(bundle exec #{command})
  else
    sh %(chef exec #{command})
  end
end

task :destroy_all do
  Rake::Task[:destroy_machines].invoke
  run_command('rm -rf Gemfile.lock && rm -rf Berksfile.lock && rm -rf cookbooks/')
end

desc "Destroy machines"
task :destroy_machines do
  run_command("chef-client #{client_opts} destroy_all.rb")
end

desc "Vendor your cookbooks/"
task :berks_vendor do
  run_command('berks vendor cookbooks')
end

desc "Create Chef Key"
task :create_key do
  if not File.exist?('.chef/validator.pem')
    require 'openssl'
    File.binwrite('.chef/validator.pem', OpenSSL::PKey::RSA.new(2048).to_pem)
  end
end

desc "All-in-One build"
task :allinone => :create_key do
  run_command("chef-client #{client_opts} vagrant_linux.rb allinone.rb")
end

desc "Multi-Node build"
task :multi_node => :create_key do
  run_command("chef-client #{client_opts} vagrant_linux.rb multi-node.rb")
end

desc "Blow everything away"
task clean: [:destroy_all]

# CI tasks
require 'rubocop/rake_task'
desc 'Run RuboCop'
RuboCop::RakeTask.new(:rubocop)

desc "Validate data bags, environments and roles"
task :json_check do
  require 'json'
  ['data_bags/*', 'environments', 'roles'].each do |sub_dir|
    Dir.glob(sub_dir + '/*.json') do |env_file|
      puts "Checking #{env_file}"
      JSON.parse(File.read(env_file))
    end
  end
end

# Helper for running various testing commands
def _run_commands(desc, commands, openstack=true)
  puts "## Running #{desc}"
  commands.each do |command, options|
    options.each do |option|
      if openstack
        sh %(sudo bash -c '. /root/openrc && #{command} #{option}')
      else
        sh %(#{command} #{option})
      end
    end
  end
  puts "## Finished #{desc}"
end

# use the correct environment depending on platform
if File.exist?('/usr/bin/apt-get')
  @platform = 'ubuntu14'
elsif File.exist?('/usr/bin/yum')
  @platform = 'centos7'
end

# Helper for looking at the starting environment
def _run_env_queries # rubocop:disable Metrics/MethodLength
    _run_commands('basic common env queries', {
      'uname' => ['-a'],
      'pwd' => [''],
      'env' => ['']},
      false
    )
  case @platform
  when 'ubuntu14'
    _run_commands('basic debian env queries', {
      'ifconfig' => [''],
      'cat' => ['/etc/apt/sources.list']},
      false
    )
  when 'centos7'
    _run_commands('basic rhel env queries', {
      '/usr/sbin/ip' => ['addr'],
      'cat' => ['/etc/yum.repos.d/*']},
      false
    )
  end
end

# Helper for setting up basic query tests
def _run_basic_queries # rubocop:disable Metrics/MethodLength
  _run_commands('basic common test queries', {
      'curl' => ['-v http://localhost', '-kv https://localhost'],
      'sudo netstat' => ['-nlp'],
      'nova-manage' => ['version', 'db version'],
      'nova' => %w(--version service-list hypervisor-list image-list flavor-list),
      'glance-manage' => %w(db_version),
      'glance' => %w(--version image-list),
      'keystone-manage' => %w(db_version),
      'keystone' => %w(--version user-list endpoint-list role-list service-list tenant-list),
      'cinder-manage' => ['version list', 'db version'],
      'cinder' => %w(--version list),
      'heat-manage' => ['db_version', 'service list'],
      'heat' => %w(--version stack-list),
      'neutron' => %w(agent-list ext-list net-list port-list subnet-list quota-list),
      'ovs-vsctl' => %w(show) }
    )
  case @platform
  when 'ubuntu14'
    _run_commands('basic debian test queries', {
      'rabbitmqctl' => %w(cluster_status),
      'ifconfig' => ['']}
    )
  when 'centos7'
    _run_commands('basic rhel test queries', {
      '/usr/sbin/rabbitmqctl' => %w(cluster_status),
      '/usr/sbin/ip' => ['addr']}
    )
  end
end

# Helper for setting up basic nova tests
def _run_nova_tests(pass) # rubocop:disable Metrics/MethodLength
  _run_commands('cinder storage volume create', {
    'cinder' => ['list', "create --display_name test_volume_#{pass} 1"],
    'sleep' => ['10'] }
  )
 _run_commands('cinder storate volume query', {
    'cinder' => ['list'] }
  )
  uuid = `sudo bash -c ". /root/openrc && cinder list | grep test_volume_#{pass} | cut -d ' ' -f 2"`
  _run_commands('nova server create', {
    'nova' => ['list', "boot test --image cirros --flavor 1 --block-device-mapping vdb=#{uuid.chomp!}:::1"],
    'sleep' => ['40'] }
  )
  _run_commands('nova server cleanup', {
    'nova' => ['list', 'show test', 'delete test'],
    'sleep' => ['25'],
    'cinder' => ['list'] }
  )
 _run_commands('nova server query', {
    'nova' => ['list'] }
  )
end

# Helper for setting up neutron local network
def _setup_local_network # rubocop:disable Metrics/MethodLength
  _run_commands('neutron local network setup', {
    'neutron' => ['net-create local_net --provider:network_type local --shared',
                  'subnet-create local_net --name local_subnet 192.168.1.0/24'] }
  )
end

# Helper for setting up tempest and upload the default cirros image. Tempest
# itself is not yet used for integration tests.
def _setup_tempest(client_opts)
  sh %(sudo chef-client #{client_opts} -E allinone-#{@platform} -r 'recipe[openstack-integration-test::setup]')
end

def _save_logs(prefix, log_dir)
  sh %(sleep 25)
  %w(nova neutron keystone cinder glance heat apache2 rabbitmq mysql openvswitch mariadb).each do |project|
    sh %(mkdir -p #{log_dir}/#{prefix}/#{project})
    sh %(sudo cp -r /etc/#{project} #{log_dir}/#{prefix}/#{project}/etc || true)
    sh %(sudo cp -r /var/log/#{project} #{log_dir}/#{prefix}/#{project}/log || true)
  end
end

desc "Integration test on Infra"
task :integration => [:create_key, :berks_vendor] do
  log_dir = ENV['WORKSPACE']+'/logs'
  # This is a workaround for allowing chef-client to run in local mode
  sh %(sudo mkdir /etc/chef && sudo cp .chef/encrypted_data_bag_secret /etc/chef/openstack_data_bag_secret)
  _run_env_queries

  # Three passes to make sure of cookbooks idempotency
  for i in 1..3
    begin
    puts "####### Pass #{i}"
    # Kick off chef client in local mode, will converge OpenStack right on the gate job "in place"
    sh %(sudo chef-client #{client_opts} -E allinone-#{@platform} -r 'role[allinone]')
    if i == 1
      _setup_tempest(client_opts)
      _setup_local_network
    end
    _run_basic_queries
    _run_nova_tests(i)

    rescue => e
      raise "####### Pass #{i} failed with #{e.message}"
    ensure
      # make sure logs are saved, pass or fail
      _save_logs("pass#{i}", log_dir)
      sh %(sudo chown -R $USER #{log_dir})
    end
  end
  # Run the tempest formal tests, setup with the openstack-integration-test cookbook
   Dir.chdir('/opt/tempest') do
     sh %(sudo -H ./run_tempest.sh --smoke --serial)
   end
end
