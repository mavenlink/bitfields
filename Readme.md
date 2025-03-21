Save migrations and columns by storing multiple booleans in a single integer.<br/>
e.g. true-false-false = 1, false-true-false = 2,  true-false-true = 5 (1,2,4,8,..)

```ruby
class User < ActiveRecord::Base
  include Bitfields
  bitfield :my_bits, 1 => :seller, 2 => :insane, 4 => :sensible
end

user = User.new(seller: true, insane: true)
user.seller # => true
user.sensible? # => false
user.my_bits # => 3
```

 - records bitfield_changes `user.bitfield_changes # => {"seller" => [false, true], "insane" => [false, true]}` (also `seller_was` / `seller_change` / `seller_changed?` / `seller_became_true?` / `seller_became_false?`)
   - Individual added methods (i.e, `seller_was`, `seller_changed?`, etc..) can be deactivated with `bitfield ..., added_instance_methods: false`
   - **Note**: ActiveRecord 5.2 changes the behavior of `_was` and `_changed?` methods when used in the context of an `after_save` callback.
     - ActiveRecord 5.1 will use the use the values that were _just_ changed.
     - ActiveRecord 5.2, however, will return the current value for `_was` and `false` for `_changed?` since the previous changes have been persisted.
 - adds scopes `User.seller.sensible.first` (deactivate with `bitfield ..., scopes: false`)
 - builds sql `User.bitfield_sql(insane: true, sensible: false) # => '(users.my_bits & 6) = 1'`
 - builds sql with OR condition `User.bitfield_sql({ insane: true, sensible: true }, query_mode: :bit_operator_or) # => '(users.my_bits & 2) = 2 OR (users.bits & 4) = 4'`
 - builds index-using sql with `bitfield ... , query_mode: :in_list` and `User.bitfield_sql(insane: true, sensible: false) # => 'users.my_bits IN (2, 3)'` (2 and 1+2) often slower than :bit_operator sql especially for high number of bits
 - builds update sql `User.set_bitfield_sql(insane: true, sensible: false) == 'my_bits = (my_bits | 6) - 4'`
 - **faster sql than any other bitfield lib** through combination of multiple bits into a single sql statement
 - gives access to bits `User.bitfields[:my_bits][:sensible] # => 4`

Install
=======

```
gem install bitfields
```

### Migration
ALWAYS set a default, bitfield queries will not work for NULL

```ruby
t.integer :my_bits, default: 0, null: false
# OR
add_column :users, :my_bits, :integer, default: 0, null: false
```

Instance Methods
================

### Global Bitfield Methods
| Method Name        | Example (`user = User.new(seller: true, insane: true`)  | Result                                                      |
|--------------------|---------------------------------------------------------|-------------------------------------------------------------|
| `bitfield_values`  | `user.bitfield_values`                                  | `{"seller" => true, "insane" => true, "sensible" => false}` |
| `bitfield_changes` | `user.bitfield_changes`                                 | `{"seller" => [false, true], "insane" => [false, true]}`    |

### Individual Bit Methods
#### Model Getters / Setters
| Method Name    | Example (`user = User.new`) | Result  |
|----------------|-----------------------------|---------|
| `#{bit_name}`  | `user.seller`               | `false` |
| `#{bit_name}=` | `user.seller = true`        | `true`  |
| `#{bit_name}?` | `user.seller?`              | `true`  |

#### Dirty Methods:

Some, not all, [`ActiveRecord::AttributeMethods::Dirty`](https://api.rubyonrails.org/v5.1.7/classes/ActiveRecord/AttributeMethods/Dirty.html) and [`ActiveModel::Dirty`](https://api.rubyonrails.org/v5.1.7/classes/ActiveModel/Dirty.html) methods can be used on each bitfield:

##### Before Model Persistence
| Method Name                        | Example (`user = User.new`)        | Result          |
|------------------------------------|------------------------------------|-----------------|
| `#{bit_name}_was`                  | `user.seller_was`                  | `false`         |
| `#{bit_name}_in_database`          | `user.seller_in_database`          | `false`         |
| `#{bit_name}_change`               | `user.seller_change`               | `[false, true]` |
| `#{bit_name}_change_to_be_saved`   | `user.seller_change_to_be_saved`   | `[false, true]` |
| `#{bit_name}_changed?`             | `user.seller_changed?`             | `true`          |
| `will_save_change_to_#{bit_name}?` | `user.will_save_change_to_seller?` | `true`          |
| `#{bit_name}_became_true?`         | `user.seller_became_true?`         | `true`          |
| `#{bit_name}_became_false?`        | `user.seller_became_false?`        | `false`         |


##### After Model Persistence
| Method Name                    | Example (`user = User.create(seller: true)`)      | Result          |
|--------------------------------|---------------------------------------------------|-----------------|
| `#{bit_name}_before_last_save` | `user.seller_before_last_save`                    | `false`         |
| `saved_change_to_#{bit_name}`  | `user.saved_change_to_seller`                     | `[false, true]` |
| `saved_change_to_#{bit_name}?` | `user.saved_change_to_seller?`                    | `true`          |

  - **Note**: These methods are dynamically defined for each bitfield, and function separately from the real `ActiveRecord::AttributeMethods::Dirty`/`ActiveModel::Dirty` methods. As such, generic methods (e.g. `attribute_before_last_save(:attribute)`) will not work.

Examples
========
Update all users

```ruby
User.seller.not_sensible.update_all(User.set_bitfield_sql(seller: true, insane: true))
```

Delete the shop when a user is no longer a seller

```ruby
before_save :delete_shop, if: -> { |u| u.seller_change == [true, false] }
```

List fields and their respective values

```ruby
user = User.new(insane: true)
user.bitfield_values(:my_bits) # => { seller: false, insane: true, sensible: false }
```

TIPS
====
 - [Upgrading] in version 0.2.2 the first field(when not given as hash) used bit 2 -> add a bogus field in first position
 - [Defaults] afaik it is not possible to have some bits true by default (without monkeypatching AR/see [tests](https://github.com/grosser/bitfields/commit/2170dc546e2c4f1187089909a80e8602631d0796)) -> choose a good naming like `xxx_on` / `xxx_off` to use the default 'false'
 - Never do: "#{bitfield_sql(...)} AND #{bitfield_sql(...)}", merge both into one hash
 - bit_operator is faster in most cases, use query_mode: :in_list sparingly
 - Standard mysql integer is 4 byte -> 32 bitfields
 - If you are lazy or bad at math you can also do `bitfields :bits, :foo, :bar, :baz`
 - If you are want more readability and reduce clutter you can do `bitfields 2**0 => :foo, 2**1 => :bar, 2**32 => :baz`

Query-mode Benchmark
=========
The `query_mode: :in_list` is slower for most queries and scales miserably with the number of bits.<br/>
*Stay with the default query-mode*. Only use :in_list if your edge-case shows better performance.

![performance](http://chart.apis.google.com/chart?chtt=bit-operator+vs+IN+--+with+index&chd=s:CEGIKNPRUW,DEHJLOQSVX,CFHKMPSYXZ,DHJMPSVYbe,DHLPRVZbfi,FKOUZeinsx,FLQWbglqw2,HNTZfkqw19,BDEGHJLMOP,BDEGIKLNOQ,BDFGIKLNPQ,BDFGILMNPR,BDFHJKMOQR,BDFHJLMOQS,BDFHJLNPRT,BDFHJLNPRT&chxt=x,y&chxl=0:|100K|200K|300K|400K|500K|600K|700K|800K|900K|1000K|1:|0|1441.671ms&cht=lc&chs=600x500&chdl=2bits+%28in%29|3bits+%28in%29|4bits+%28in%29|6bits+%28in%29|8bits+%28in%29|10bits+%28in%29|12bits+%28in%29|14bits+%28in%29|2bits+%28bit%29|3bits+%28bit%29|4bits+%28bit%29|6bits+%28bit%29|8bits+%28bit%29|10bits+%28bit%29|12bits+%28bit%29|14bits+%28bit%29&chco=0000ff,0000ee,0000dd,0000cc,0000bb,0000aa,000099,000088,ff0000,ee0000,dd0000,cc0000,bb0000,aa0000,990000,880000)

Testing With RSpec
=========

To assert that a specific flag is a bitfield flag and has the `active?`, `active`, and `active=` methods and behavior use the following matcher:

````ruby
require 'bitfields/rspec'

describe User do
  it { should have_a_bitfield :active }
end
````

TODO
====
 - convenient named scope `User.with_bitfields(xxx: true, yyy: false)`

Authors
=======
### [Contributors](http://github.com/grosser/bitfields/contributors)
 - [Hellekin O. Wolf](https://github.com/hellekin)
 - [John Wilkinson](https://github.com/jcwilk)
 - [PeppyHeppy](https://github.com/peppyheppy)
 - [kmcbride](https://github.com/kmcbride)
 - [Justin Aiken](https://github.com/JustinAiken)
 - [szTheory](https://github.com/szTheory)
 - [Reed G. Law](https://github.com/reedlaw)
 - [Rael Gugelmin Cunha](https://github.com/reedlaw)
 - [Alan Wong](https://github.com/naganowl)
 - [Andrew Bates](https://github.com/a-bates)
 - [Shirish Pampoorickal](https://github.com/shirish-pampoorickal)

[Michael Grosser](http://grosser.it)<br/>
michael@grosser.it<br/>
License: MIT<br/>
[![Build Status](https://travis-ci.org/grosser/bitfields.png)](https://travis-ci.org/grosser/bitfields)
