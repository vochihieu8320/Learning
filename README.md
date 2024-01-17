# Learning

<h1>Insert large record </h1>
There is a result of my testing inserting 1 million records to DB. The question is how to insert these large records
<h2>Insert by batch size</h2>

```
bach_size = 10000
limit_record = 1_000_000

client = Client. first
infections = []

time = Benchmark.measure do
    (1..limit_record).each do |i|
        infections << { client_id: client.id, clinic_id: 1, value: 1 }

        if infections.length == bach_size
            puts "write #{bach_size}"
            xx.insert_all!(infections)
            infections = []
        end
    end

    xx.insert_all!(infections) if infections.length > 0
end

puts "Execution time: #{time.real} seconds"
```

The result would be 29.77178999991156 seconds. This is not bad. But if the client said: Hey, Do you have another way to do it? Do not answer them let increase the batch size 😄.
Now let's dive into other solution

<h2>Create new thread to insert </h2>

```
bach_size = 100000
limit_record = 1_000_000
client = Client.first
infections = []
time = Benchmark.measure do
    threads = []

    (1..limit_record).each do |i|
        infections << { client_id: client.id, clinic_id: 1, value: 1 }

        if infections.length == bach_size
            threads << Thread.new(infections.dup) do |batch|
                puts "write #{bach_size}"
                Infection.insert_all!(batch)
            end
            infections = []
        end
    end

    threads.each(&:join)
    Infection.insert_all!(infections) if infections.length > 0
end

puts "Execution time: #{time.real} seconds"
```

You know when I try this I think we trade off your CPU to achieve it. Please consider it 
Number thread 10:
execute time: 26.462406999897212
It is just a little faster


<h1>Read 1GBcsv file without using load data infile </h1>

<h2>Read by batch size</h2>

I'm using for each and read line by line into batch size. But I think it is not suitable because of it. I dont know when it finish

<h2>Split into smaller file and then eat it</h2
    def split_csv_file(file_path, batch_size)
      file_number = 1
      row_count = 0
      output_file = nil
      Dir.mkdir("temp")
    
      CSV.foreach(file_path, headers: true) do |row|
        if row_count % batch_size == 0
          puts "write #{row_count} records to temp/output_#{file_number}.csv"
          output_file = File.open("temp/output_#{file_number}.csv", 'w')
          output_file.puts(row.headers.join(',')) if row_count == 0
          file_number += 1
        end
    
        output_file.puts(row.fields.join(','))
    
        row_count += 1
      end
    
      output_file.close if output_file
    end

    batch_size = 100_000
    limit_record = 1_000_000
    
    file_path = 'sample.csv'
    client = Client.first
    infections = []
    
    memory_command = "ps -o rss= -p #{Process.pid}"
    memory_before = %x(#{memory_command}).to_i
    time = Benchmark.realtime do
      split_csv_file(file_path, batch_size)
      csv_files = Dir.glob(File.join("temp", '*.csv'))
    
      csv_files.each do |csv_file|
        values = []
        puts "entry", csv_file
    
        CSV.foreach(csv_file, headers: true).with_index do |row, index|
          # Process each row of the CSV file
          values << { value: index }
    
          if values.length == batch_size
            puts "write #{csv_file} to DB"
            Test.insert_all!(values)
            values = []
          end
        end
      end
    end
    
    puts "Time: #{time.round(2)}"
    
    memory_after = %x(#{memory_command}).to_i
    puts "Memory: #{((memory_after - memory_before) / 1024.0).round(2)} MB"
```

For reading file into DB
Time: 204.59
Memory: 113.53 MB

To split 1gb file into smaller file
Time: 237.68
Memory: 3.23 MB

```
<h1>Load 1 gb file csv using load local infile</h1>

    require 'benchmark'
    require 'csv'
    
    def split_csv_file(file_path, batch_size)
      file_number = 1
      row_count = 0
      output_file = nil
      Dir.mkdir("temp")
    
      CSV.foreach(file_path, headers: true) do |row|
        if row_count % batch_size == 0
          puts "write #{row_count} records to temp/output_#{file_number}.csv"
          output_file = File.open("temp/output_#{file_number}.csv", 'w')
          output_file.puts(row.headers.join(',')) if row_count == 0
          file_number += 1
        end
    
        output_file.puts(row.fields.join(','))
    
        row_count += 1
      end
    
      output_file.close if output_file
    end
    
    batch_size = 100_000
    limit_record = 1_000_000
    
    file_path = 'sample.csv'
    client = Client.first
    infections = []
    
    memory_command = "ps -o rss= -p #{Process.pid}"
    memory_before = %x(#{memory_command}).to_i
    time = Benchmark.realtime do
      split_csv_file(file_path, batch_size)
      csv_files = Dir.glob(File.join("temp", '*.csv'))
      csv_files.each do |csv_file|
        values = []
        puts "entry", csv_file
        sql = <<-SQL
          LOAD DATA LOCAL INFILE "#{csv_file}"
          INTO TABLE test
          FIELDS TERMINATED BY ',' ENCLOSED BY '"'
          LINES TERMINATED BY '\n'
          IGNORE 1 ROWS;
        SQL
    
        ActiveRecord::Base.connection.execute(sql)
      end
    end
    
    puts "Time: #{time.round(2)}"
    memory_after = %x(#{memory_command}).to_i
    puts "Memory: #{((memory_after - memory_before) / 1024.0).round(2)} MB"
```

For reading file into DB
Time: 76.36
Memory: 10.49 MB

To split 1gb file into smaller file
Time: 237.68
Memory: 3.23 MB

```
<h1>Install sidekiq</h1>
<h2>Add gem</h2>

```
gem 'sidekiq'
gem 'sidekiq-scheduler'
```
<h2>Add config/sidekiq_schedule.yml</h2>

```
every_minute_job:
  cron: "5 * * * *"   # Runs every minute
  class: "MyWorker"   # Your Sidekiq worker class
```

<h2>Add sidekiq initalizer</h2>

```
Sidekiq.configure_server do |config|
  config.redis = { url: 'redis://localhost:6379/0' }
  config.on(:startup) do
    Sidekiq.schedule = YAML.load_file(File.expand_path('../../sidekiq_schedule.yml', __FILE__))
    Sidekiq::Scheduler.reload_schedule!
  end
end
```

Finally, create worker app/workers/...
