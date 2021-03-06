#課題内容(ルータの CLI を作ろう)
>
>ルータのコマンドラインインタフェース (CLI) を作ろう。
>
>次の操作ができるコマンドを作ろう。
>
>* ルーティングテーブルの表示
>* ルーティングテーブルエントリの追加と削除
>* ルータのインタフェース一覧の表示
>* そのほか、あると便利な機能
>
>コントローラを操作するコマンドの作りかたは、第3回パッチパネルで作った patch_panel コマンドを参考にしてください。

#解答
##0. コマンドの仕様
今回の課題では，実行用のバイナリとして/bin/simple_routerを用意し，要求されたコマンドはこのバイナリに引数としてサブコマンドを与えて実行するものとした．また、コマンドの実行結果を表示する際は，Trema runプロセスが実行されている端末ではなく，このバイナリが実行される端末に表示するものとした．

各サブコマンドの仕様は以下に定めた．

1. ルーティングテーブルの表示
  * 使用方法: `/bin/simple_router show_routing_table`
1. ルーティングテーブルエントリの追加
  * 使用方法: `/bin/simple_router add_entry [宛先ip] [ネットマスク] [転送先]`
  * 宛先ip、転送先は8ビットずつ10進表記
  * ネットマスクは数値で指定
1. ルーティングテーブルエントリの削除
  * 使用方法: `/bin/simple_router del_entry [宛先ip] [ネットマスク]`
  * 宛先ipは8ビットずつ10進表記
  * ネットマスクは数値で指定
1. ルーターのインターフェース一覧の表示
  * 使用方法: `/bin/simple_router show_interface`

##1. ルーティングテーブルの表示
###1.1 コマンドの設計方針
取得したいルーティングテーブルの情報はRoutingTableクラス(/lib/routing_table.rbで実装)のインスタンス変数@dbで管理されているので、この情報を表示するようにコマンドを設計する．  
RoutingTableクラスのインスタンスはTrema runプロセスによって起動されるSimpleRouterクラスのインスタンス(/lib/simple_router.rbで実装)がインスタンス変数@routing_tableとして管理しているので、以下の処理を各ファイルに実装した．

* `/bin/simple_router`   : SimpleRouter#show_RT()を呼び出すコマンドshow_routing_tableを実装
* `/lib/simple_router.rb`: RoutingTable#list()を呼び出すメソッドshow_RT()を実装
* `/lib/routing_table.rb`: @dbとネットマスクの最大値を返すメソッドlist()を実装

コマンド実行時の呼び出し関係を以下の図に示す．

![図１](https://github.com/handai-trema/simple-router-d-miura/blob/master/fig1.png)

###1.2 コマンドの実装内容
###1.2.1 `/bin/simple_router`(対応部分のみ抜粋)  
* IPv4Addressクラスを利用するため、Pioをincludeした．  
```ruby
include Pio
```
* SimpleRouter#show_RT()で取得したルーティングテーブルの内容を表示する処理を記述した．
```ruby
desc 'List the Routing Table'
command :show_routing_table do |c|
  c.desc 'Location to find socket files'
  c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR
  c.action do |_global_options, options, args|

    @db, @length = Trema.trema_process('SimpleRouter', options[:socket_dir]).controller.
      show_RT()

    print "destination \t\t  next hop\n"

    @length.downto(0).each do |eachMask|
      @db[eachMask].each do |dest, next_hop|
        print IPv4Address.new(dest).to_s+"/"+eachMask.to_s+"\t\t"+next_hop.to_s+"\n"
      end
    end

  end
end
```

###1.2.2 `/lib/simple_router.rb`(追加部分のみ抜粋)  
* RoutingTableクラスのインスタンス変数@routing_tableのインスタンスメソッドlistの結果を返すメソッドshow_RTを追加した．
```ruby
def show_RT()
  logger.info "show_routing_table() is called"
  return @routing_table.list()
end
```
###1.2.3 `/lib/routing_table.rb`(追加部分のみ抜粋)  
* ルーティングテーブルの内容であるインスタンス変数@dbとネットマスクの最大長を返すメソッドlistを追加した
```ruby
def list()
  return @db, MAX_NETMASK_LENGTH
end
```
###1.3 実行結果
デフォルトの設定ファイル/simple_router.confでルーターを起動し，コマンドshow_routing_tableを実行した．
```
  ROUTES = [
    {
      destination: '0.0.0.0',
      netmask_length: 0,
      next_hop: '192.168.1.2'
    }
  ]
```
実行結果は以下のようになった
```
$ ./bin/simple_router show_routing_table
destination 		  next hop
0.0.0.0/0		192.168.1.2
```


##2. ルーティングテーブルエントリの追加と削除
###2.1 コマンドの設計方針
エントリの追加に関しては、RoutingTableクラスにこの処理を行うメソッドaddがすでにソースコード(/lib/routing_table.rb)に記載されているので、このメソッドを自作バイナリから呼び出す処理を追加した．また、エントリの削除もこのメソッドを参考にして追加した．

コマンド実行時の呼び出し関係を以下の図に示す．

![図2](https://github.com/handai-trema/simple-router-d-miura/blob/master/fig2.png)

###2.2 コマンドの実装内容
###2.2.1 `/bin/simple_router`(対応部分のみ抜粋)
* 宛先ipと転送先は文字列、ネットマスクは整数でSimpleRouter#add_routing_table/del_routing_tableを呼び出している．
```ruby
desc 'Add Entry to the Routing Table'
arg_name 'destination_ip, netmask, forward_to'
command :add_entry do |c|
  c.desc 'Location to find socket files'
  c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR

  c.action do |_global_options, options, args|
    destination_ip = args[0]
    netmask = args[1].to_i
    next_hop = args[2]
    Trema.trema_process('SimpleRouter', options[:socket_dir]).controller.
      add_routing_table(destination_ip, netmask, next_hop)
  end
end

desc 'Delete Entry to the Routing Table'
arg_name 'destination_ip, netmask, forward_to'
command :del_entry do |c|
  c.desc 'Location to find socket files'
  c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR

  c.action do |_global_options, options, args|
    destination_ip = args[0]
    netmask = args[1].to_i
    Trema.trema_process('SimpleRouter', options[:socket_dir]).controller.
      del_routing_table(destination_ip, netmask)
  end
end
```
###2.2.2 `/lib/simple_router.rb`(追加部分のみ抜粋)
* RoutingTableクラスのインスタンスメソッドadd/delを呼び出している．
```ruby
def add_routing_table(destination_ip, netmask, next_hop)
  logger.info "add_routing_table() is called"
  @routing_table.add({:destination => destination_ip, :netmask_length => netmask, :next_hop => next_hop})
end

def del_routing_table(destination_ip, netmask)
  logger.info "del_routing_table() is called"
  @routing_table.del({:destination => destination_ip, :netmask_length => netmask})
end
```
###2.2.3 `/lib/routing_table.rb`(追加部分のみ抜粋)
* ルーティングテーブルエントリを追加するメソッドaddはすでに定義されていたので、削除を行うメソッドdelを追加した．
```ruby
def del(options)
  netmask_length = options.fetch(:netmask_length)
  prefix = IPv4Address.new(options.fetch(:destination)).mask(netmask_length)
  @db[netmask_length].delete(prefix.to_i)
end
```
###2.3 コマンドの実行結果
エントリの追加と削除を行い，逐次ルーティングテーブルのエントリを表示させることで動作の確認を行った．
実行結果を以下に示す．
```
$ ./bin/simple_router show_routing_table
destination 		  next hop
0.0.0.0/0		192.168.1.2
$ ./bin/simple_router add_entry 192.168.1.10 24 192.168.1.2
$ ./bin/simple_router show_routing_table
destination 		  next hop
192.168.1.0/24		192.168.1.2
0.0.0.0/0		192.168.1.2
$ ./bin/simple_router add_entry 192.168.1.10 16 192.168.1.2
$ ./bin/simple_router show_routing_table
destination 		  next hop
192.168.1.0/24		192.168.1.2
192.168.0.0/16		192.168.1.2
0.0.0.0/0		192.168.1.2
$ ./bin/simple_router del_entry 192.168.1.10 24
$ ./bin/simple_router show_routing_table
destination 		  next hop
192.168.0.0/16		192.168.1.2
0.0.0.0/0		192.168.1.2
```


##3. ルーターのインターフェース一覧の表示
###3.1 コマンドの設計方針
インターフェースは./lib/interface.rbに実装されているInterfaceクラスのインスタンスとして管理されている．
そこで、コントローラが管理するInterfaceクラスのオブジェクトを返すメソッドshow_IFをSimpleRouterクラスに実装し、
自作バイナリではこのメソッドを呼び出して得られたオブジェクトの内容を表示すした．

コマンド実行時の呼び出し関係を以下の図に示す．

![図3](https://github.com/handai-trema/simple-router-d-miura/blob/master/fig3.png)

###3.2 コマンドの実装内容
###3.2.1 `/bin/simple_router`(対応部分のみ抜粋)
* Interfaceクラスのソースコードを相対パスで指定し、読み込む
```ruby
require './lib/interface'
```
* Interfaceクラスのオブジェクトの内容を表示
```ruby
desc 'List the Interface of Router'
command :show_interface do |c|
  c.desc 'Location to find socket files'
  c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR
  c.action do |_global_options, options, args|
    interfaces = Trema.trema_process('SimpleRouter', options[:socket_dir])
                      .controller
                      .show_IF()
    print "port_number\tmac_address\t\tip_address/netmask\n"
    interfaces.each do |each|
      print each.port_number.to_s+"\t\t"+each.mac_address.to_s+"\t"+each.ip_address.to_s+"/"+each.netmask_length.to_s+"\n"
    end
  end
end
```
###3.2.2 `/lib/simple_router.rb`(追加部分のみ抜粋)
* Interface.allで取得したInterfaceクラスのすべてのインスタンスを返している．
```ruby
def show_IF()
  logger.info "show_interface() is called"
  return Interface.all
end
```
###3.3 コマンドの実行結果
デフォルトの設定ファイル/simple_router.confでルーターを起動し，コマンドshow_routing_tableを実行した．
```
INTERFACES = [
  {
    port: 1,
    mac_address: '01:01:01:01:01:01',
    ip_address: '192.168.1.1',
    netmask_length: 24
  },
  {
    port: 2,
    mac_address: '02:02:02:02:02:02',
    ip_address: '192.168.2.1',
    netmask_length: 24
  }
]
```
実行結果は以下のようになった
```
$ ./bin/simple_router show_interface
port_number	mac_address		ip_address/netmask
1		01:01:01:01:01:01	192.168.1.1/24
2		02:02:02:02:02:02	192.168.2.1/24
```
