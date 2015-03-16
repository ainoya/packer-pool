@access_key = ENV['AWS_ACCESS_KEY_ID']
@secret_key = ENV['AWS_SECRET_ACCESS_KEY']
@region     = ENV['AWS_DEFAULT_REGION']
@subnet_id  = ENV['AWS_PACKER_SUBNET_ID']
@security_group_id = ENV['AWS_PACKER_SG_ID']

task :plan => :cloud_config do
  Dir.chdir("terraform") do
    sh %Q(terraform plan -input=false -var "aws_access_key=#{@access_key}" -var "aws_secret_key=#{@secret_key}" -out plan)
  end
end

task :apply => :plan do
  @ami_id = File.readlines('ami-id').first.chomp
  Dir.chdir("terraform") do
    sh %Q(terraform apply -input=false -var "aws_access_key=#{@access_key}" -var "aws_secret_key=#{@secret_key}" -var "ami=#{@ami_id}" < plan)
    puts @discovery_url
  end
end

task :cloud_config => :discovery_url do
	sh %Q(cat terraform/userdata/core-userdata.template | sed -e "s\#{{ discovery_url }}##{@discovery_url}#" -e "s\#{{ purpose_role }}#general#" > terraform/userdata/core-userdata)
end

task :clean do
	sh "rm -f terraform/userdata/core-userdata" 
end

task :coreos_hvm_ami_id do
  json = `curl -s http://stable.release.core-os.net/amd64-usr/current/coreos_production_ami_all.json`
	@coreos_ami_id=`echo '#{json}' | jq -r '.amis[] | select(.name == \"#{@region}\") | .hvm'`.chomp
  puts @coreos_ami_id
end

task :ami => :coreos_hvm_ami_id do
  Dir.chdir("packer") do
    sh %Q(cd packer && packer build -machine-readable \
          -var "aws_access_key=#{@access_key}" \
          -var "aws_secret_key=#{@secret_key}" \
          -var "source_ami=#{@coreos_ami_id}" \
          -var "subnet_id=#{@subnet_id}" \
          -var "security_group_id=#{@security_group_id}" \
          -var "region=#{@region}" core.json | tee ../../build.log)
  end

	@ami_id=`grep 'artifact,0,id' build.log | cut -d, -f6 | cut -d: -f2`.chomp
  File.write('ami-id', @ami_id)
  puts @ami_id
end

task :all => [:ami, :apply]
