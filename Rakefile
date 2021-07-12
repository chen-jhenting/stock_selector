# frozen_string_literal: false

require 'net/http'
require 'JSON'
require 'date'
require 'fileutils'

# 有參考意義的會打勾

FILE_TABLE = {
  'primary_stock_data.txt' => '收集近 60 交易日股票資訊',
  'first_step_filter.txt' => '依當日成交量與股價上下限篩選的資料',
  'second_step_filter.txt' => '依近五日成交量最低值與平均值篩選的資料',
  'third_step_filter.txt' => '依 當日股價 > ma5 > ma20 > ma60 篩選的資料',
  'third_step_filter_stock_symbols.txt' => '擷取 third_step_filter.txt 股票代號與公司名稱', # v
  'fourth_step_filter.txt' => '當日股價 帶量 突破 20 日高點',
  'fourth_step_filter_stock_symbols.txt' => '擷取 fourth_step_filter.txt 股票代號與公司名稱', # v
  'fifth_step_filter.txt' => '剛起漲的股票',
  'fifth_step_filter_stock_symbols.txt' => '擷取 fifth_step_filter.txt 股票代號與公司名稱' # v
}.freeze

CONDUCT_ORDER = {
  query_stock_info: 1,
  first_step_filter: 2,
  second_step_filter: 3,
  third_step_filter: 4,
  fourth_step_filter: 5,
  fifth_step_filter: 6
}.freeze

# data structure
# {
#   'stock_symbol' => {
#     'company_name' => 'xx',
#     'price' => [1, 2, 3, 4],
#     'amount' => [1, 2, 3, 4],
#     'ma5' => 123,
#     'ma20' => 555,
#     'ma60' => 1111,
#   },
# }

#-------------------------------------------------------------------------------------------------------
desc '收集近 60 天股價資訊'
task :query_stock_info do
  puts '開始收集股票資料'
  working_days = []
  stock_data = {}
  date = Date.today
  while working_days.size <= 60
    puts "目前還需要 #{60 - working_days.size} 筆資料"
    query_date = date.strftime('%Y%m%d')
    raw_data = query_data_from_twse(query_date)

    if raw_data
      puts "開始處理 #{query_date} 股票資料"
      raw_data.each do |data|
        stock_symbol, company_name, amount, price = extract_used_value(data)
        next unless stock_symbol =~ /\A[^0]\d{3}$/

        arranged_data_based_on_stock_symbol(stock_data, stock_symbol, company_name, amount, price)
      end
      working_days << query_date
    else
      puts "#{query_date} 當天並無資料"
    end
    date = date.prev_day(1)
    sleep(3)
  end
  count_average_value(stock_data)

  File.open('primary_stock_data.txt', 'wb') { |f| f.write(stock_data.to_json) }
  puts '已收集近 60 天股票資料，請查閱 primary_stock_data.txt'
end

def extract_used_value(data)
  stock_symbol, company_name, amount, price = data.values_at(0, 1, 3, 8)
  amount = amount.gsub(',', '').to_i
  price = price.to_f
  [stock_symbol, company_name, amount, price]
end

def arranged_data_based_on_stock_symbol(stock_data, stock_symbol, company_name, amount, price)
  stock_data[stock_symbol] ||= {}
  stock_data[stock_symbol]['company_name'] ||= company_name
  stock_data[stock_symbol]['amount'] ||= []
  stock_data[stock_symbol]['amount'] << amount
  stock_data[stock_symbol]['price'] ||= []
  stock_data[stock_symbol]['price'] << price
end

def count_average_value(stock_data)
  stock_data.each do |_stock_symbol, hash|
    puts "計算 #{hash['company_name']} 平均值"
    hash['ma5'] = (hash['price'].first(5).sum / 5.0).round(2)
    hash['ma20'] = (hash['price'].first(20).sum / 20.0).round(2)
    hash['ma60'] = (hash['price'].first(60).sum / 60.0).round(2)
  end
end

#-------------------------------------------------------------------------------------------------------
desc '篩選符合 當日成交量 股價下限 股價上限 的股票，all args => number'
task :first_step_filter, %i[trade_amount lower_price_limit upper_price_limit] do |_task, args|
  trade_amount, lower_price_limit, upper_price_limit =
    [args_trade_amount(args), args_lower_price_limit(args), args_upper_price_limit(args)]

  puts "開始第一步篩選，成交量須高於#{trade_amount}，股價高於#{lower_price_limit}，股價低於#{upper_price_limit}"

  primary_stock_data = read_file('primary_stock_data.txt')
  processed_data = primary_stock_data.clone
  puts "原始資料 #{processed_data.size} 筆"

  primary_stock_data.each do |stock_symbol, stock_data|
    price = price(stock_data).first
    amount = amount(stock_data).first
    next processed_data.delete(stock_symbol) if amount < trade_amount # 成交張數限制
    next processed_data.delete(stock_symbol) unless price.between?(lower_price_limit, upper_price_limit)# 股價限制

    puts "保留 #{stock_symbol} #{stock_data['company_name']}"
  end

  File.open('first_step_filter.txt', 'wb') { |f| f.write(processed_data.to_json) }
  puts "已根據當日成交量與股價上下限篩選出 #{processed_data.size} 筆"
  puts '請查閱 first_step_filter.txt'
end

#-------------------------------------------------------------------------------------------------------
desc '篩選符合 近五日最低成交量 近五日平均成交量，all args => number'
task :second_step_filter, %i[trade_amount] do |_task, args|
  trade_amount = args_trade_amount(args)
  puts "開始第二步篩選，五日最低成交量與近五日平均成交量須高於#{trade_amount}"

  first_step_filter_data = read_file('first_step_filter.txt')
  processed_data = first_step_filter_data.clone
  puts "原始資料 #{processed_data.size} 筆"

  first_step_filter_data.each do |stock_symbol, stock_data|
    next processed_data.delete(stock_symbol) if amount(stock_data).first(5).min < trade_amount
    next processed_data.delete(stock_symbol) if (amount(stock_data).first(5).sum / 5.0) < trade_amount

    puts "保留 #{stock_symbol} #{stock_data['company_name']}"
  end

  File.open('second_step_filter.txt', 'wb') { |f| f.write(processed_data.to_json) }
  puts "已根據五日最低成交量與近五日平均成交量篩選出 #{processed_data.size} 筆"
  puts '請查閱 second_step_filter.txt'
end

#-------------------------------------------------------------------------------------------------------
desc '篩選符合 當日股價 > ma5 > ma20 > ma60'
task :third_step_filter do
  puts '開始第三步篩選，股價呈現多頭走勢，當日股價 > ma5 > ma20 > ma60'
  second_step_filter_data = read_file('second_step_filter.txt')
  processed_data = second_step_filter_data.clone
  puts "原始資料 #{processed_data.size} 筆"

  second_step_filter_data.each do |stock_symbol, stock_data|
    current_price, ma5, ma20, ma60 =
      [current_price(stock_data), ma5(stock_data), ma20(stock_data), ma60(stock_data)]

    next processed_data.delete(stock_symbol) unless current_price > ma5
    next processed_data.delete(stock_symbol) unless ma5 > ma20
    next processed_data.delete(stock_symbol) unless ma20 > ma60

    puts "保留 #{stock_symbol} #{stock_data['company_name']}"
  end

  stock_symbols = extract_stock_symbol_and_company_name(processed_data)

  File.open('third_step_filter.txt', 'wb') { |f| f.write(processed_data.to_json) }
  File.open('third_step_filter_stock_symbols.txt', 'wb') { |f| f.write(stock_symbols.to_json) }
  puts "已根據多頭走勢，當日股價 > ma5 > ma20 > ma60，篩選出 #{processed_data.size} 筆"
  puts '詳細資料請查閱 third_step_filter.txt'
  puts '單純股票代碼請查閱 third_step_filter_stock_symbols.txt'
end

#-------------------------------------------------------------------------------------------------------

desc '篩選符合 當日股價 帶量 突破 20 日高點或是突破拉回'
task :fourth_step_filter do
  puts '開始第四步篩選，當日股價 帶量 突破'
  third_step_filter_data = read_file('third_step_filter.txt')
  processed_data = third_step_filter_data.clone
  puts "原始資料 #{processed_data.size} 筆"

  third_step_filter_data.each do |stock_symbol, stock_data|
    current_price, price, current_amount, five_day_average_amount =
      [current_price(stock_data), price(stock_data), current_amount(stock_data), (amount(stock_data).first(5).sum / 5)]

    next processed_data.delete(stock_symbol) unless price.first(20).max(3).include?(current_price) # 當日股價必須是近20天的前三高，代表突破或者突破拉回
    next processed_data.delete(stock_symbol) if (current_amount / five_day_average_amount) < 1.3

    puts "保留 #{stock_symbol} #{stock_data['company_name']}"
  end

  stock_symbols = extract_stock_symbol_and_company_name(processed_data)

  File.open('fourth_step_filter.txt', 'wb') { |f| f.write(processed_data.to_json) }
  File.open('fourth_step_filter_stock_symbols.txt', 'wb') { |f| f.write(stock_symbols.to_json) }
  puts "已根據 當日股價 帶量 突破 且 開始上漲，篩選出 #{processed_data.size} 筆"
  puts '詳細資料請查閱 fourth_step_filter.txt'
  puts '單純股票代碼請查閱 fourth_step_filter_stock_symbols.txt'
end

#-------------------------------------------------------------------------------------------------------
desc '篩選符合 剛起漲'
task :fifth_step_filter do
  puts '開始第五步篩選，剛起漲'
  fourth_step_filter_data = read_file('fourth_step_filter.txt')
  processed_data = fourth_step_filter_data.clone
  puts "原始資料 #{processed_data.size} 筆"

  fourth_step_filter_data.each do |stock_symbol, stock_data|
    ma5, ma20 = [ma5(stock_data), ma20(stock_data)]

    next processed_data.delete(stock_symbol) unless (ma5 / ma20).between?(1.02, 1.05) # 週線跟月線差太遠，就代表漲一段了

    puts "保留 #{stock_symbol} #{stock_data['company_name']}"
  end

  stock_symbols = extract_stock_symbol_and_company_name(processed_data)

  File.open('fifth_step_filter.txt', 'wb') { |f| f.write(processed_data.to_json) }
  File.open('fifth_step_filter_stock_symbols.txt', 'wb') { |f| f.write(stock_symbols.to_json) }
  puts "已根據 剛起漲，篩選出 #{processed_data.size} 筆"
  puts '詳細資料請查閱 fifth_step_filter.txt'
  puts '單純股票代碼請查閱 fifth_step_filter_stock_symbols.txt'
end

#-------------------------------------------------------------------------------------------------------
desc '從收集資料，第一步篩選....到第五步篩選'
task :daily_stock_selection do
  system 'rake query_stock_info'
  system 'rake "first_step_filter[1000, 20, 100]"'
  system 'rake "second_step_filter[1000]"'
  system 'rake third_step_filter'
  system 'rake fourth_step_filter'
  system 'rake fifth_step_filter'

  date = Date.today
  time = nil
  until time
    query_date = date.strftime('%Y%m%d')
    if query_data_from_twse(query_date)
      time = query_date
    else
      sleep(3)
    end
  end

  Dir.mkdir time unless File.exist?(time)

  FILE_TABLE.each do |file, _description|
    FileUtils.move file, "#{time}/#{file}"
  end
end

#-------------------------------------------------------------------------------------------------------
# 共用 method 區

def read_file(file)
  json_data = File.read(file)
  JSON.parse(json_data)
end

def extract_stock_symbol_and_company_name(processed_data)
  processed_data.map { |stock_symbol, stock_data| { stock_symbol => stock_data['company_name'] } }
end

def args_trade_amount(args)
  args.trade_amount.to_i
end

def args_lower_price_limit(args)
  args.lower_price_limit.to_f
end

def args_upper_price_limit(args)
  args.upper_price_limit.to_f
end

def current_price(stock_data)
  stock_data['price'].first
end

def price(stock_data)
  stock_data['price']
end

def current_amount(stock_data)
  stock_data['amount'].first
end

def amount(stock_data)
  stock_data['amount']
end

def ma5(stock_data)
  stock_data['ma5']
end

def ma20(stock_data)
  stock_data['ma20']
end

def ma60(stock_data)
  stock_data['ma60']
end

def query_data_from_twse(query_date)
  uri = URI("https://www.twse.com.tw/exchangeReport/MI_INDEX?response=json&date=#{query_date}&type=ALL")
  response = Net::HTTP.get(uri)
  JSON.parse(response)['data9']
end
