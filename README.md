# Learning

<h1>Insert large record </h1>
There is a result of my testing inserting 1 million records to DB. The questions is how to insert this large records
<h2>Insert by batch size</h2>

bach_size = 10000
limit_record = 1_000_000

client = Client.first
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
