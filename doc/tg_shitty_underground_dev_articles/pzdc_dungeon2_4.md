**PZDC_dungeon_2. Добавление новой фичи, на примере опций для выбора скорости анимации. Часть 1**

До сегодня скорость анимации в бою определялась просто хардкодом при помощи `sllep(1)` между отрисовкой экранов с анимациями в методах энджена `engines\attacks_round`, но пришло время добавить опции, чтобы игрок сам мог выбрать скорость анимации. Опишу процесс по горячим следам, частично описание будет в комментариях вставленного кода.

1. Создадим модель `models\options\options`. При первом создании экземпляра, при запуске игры, будет сгенерирован yml-фаил с настройками опций по умолчанию, сейчас там будет только значение для скорости анимации, но далее можно будет добавить выбор языка или еще какие-то настройки для пользователя. В этой модели стоило бы наверно применить сингелтон, но пока так.

```ruby
class Options
  PATH = 'saves/options.yml'
  SPEEDS_FOR_ANIMATIONS = [0.1, 0.4, 0.7, 1, 1.5] # значения для скоростей анимаций

  def initialize
    create() # метод генерирует yml-фаил с опциями по умолчанию если его еще нет
    @options = YAML.safe_load_file(PATH)
  end

  # метод для применения в энджине engines\attacks_round, собственно чтобы установить там выбранную скорость анимации
  def get_enemy_actions_animation_speed
    @options['enemy_actions_animation_speed']
  end

  # метод устанавливает анимацию, принимает индекс из энджина engines\options_engine который будет описан ниже
  def set_enemy_actions_animation_speed_to(n)
    @options['enemy_actions_animation_speed'] = SPEEDS_FOR_ANIMATIONS[n-1]
    update()
  end

  # геттеры для вставок значений во views, приходится применять method_missing, опишу об этом подробнее в следующих постах с объяснением как работают вьюхи и рендереры
  def method_missing(method)
    method_name, par = method.to_s.split('__')
    if method_name == 'enemy_actions_animation_speed'
      @options['enemy_actions_animation_speed'] == SPEEDS_FOR_ANIMATIONS[par.to_i] ? '   (+)   ' : "[Enter #{par.to_i+1}]"
    end
  end

  private

  def create # метод генерирует yml-фаил с опциями по умолчанию если его еще нет
    File.write(PATH, new_file_data().to_yaml) unless RubyVersionFixHelper.file_exists?(PATH)
  end

  def update # метод обновляет значения опций в yml-фаиле
    File.write(PATH, @options.to_yaml)
  end

  def new_file_data # опции по умолчанию для начальной генерации yml-фаила
    {
      'enemy_actions_animation_speed' => 1
    }
  end
end
```

PZDC_dungeon_2 на Github https://github.com/krillan49/PZDC_dungeon_2


**PZDC_dungeon_2. Добавление новой фичи, на примере опций для выбора скорости анимации. Часть 2**

2. Создадим новый класс энджин `engines\options_engine`, который будем запускать в главном энджине(будет описано ниже). Он будет вызывать отрисовку новых экранов, принимать выбор пользователя и устанавливать значения опций в свойства модели `models\options\options`, чтобы они записались в yml-фаил

```ruby
class OptionsEngine
  def initialize
    # нужен только экземпляр модели `models\options\options`
    @options = Options.new
  end

  # основной метод отрисовывает экран всех видов опций(на будущее) из которого можно перейти на экран выбора скорости анимаций
  def main
    choose = nil
    until ['', '0'].include?(choose)
      # отрисовывает главный рендерер, его как вьюхи опишу когда-то в будущих постах
      MainRenderer.new(:options_choose_screen).display
      choose = gets.strip
      if choose == '1'
        # переходим на экран выбора скорости анимаций
        animation_speed()
      end
    end
  end

  # метод отрисовывает экран выбора анимаций и принимает выбор пользователя и передающий его в модель опций
  def animation_speed
    choose = nil
    until ['', '0'].include?(choose)
      MainRenderer.new(:options_animation_speed_screen, entity: @options).display
      choose = gets.strip
      if ('1'..'5').include?(choose)
        # передаем индекс для определения значения скорости в модель опций
        @options.set_enemy_actions_animation_speed_to(choose.to_i)
      end
    end
  end
end
```


PZDC_dungeon_2 на Github https://github.com/krillan49/PZDC_dungeon_2


**PZDC_dungeon_2. Добавление новой фичи, на примере опций для выбора скорости анимации. Часть 3**

3. В главном энджине `engines\main` в его главный метод добавим переход к опциям

```ruby
class Main

  # ...

  def start_game
    # Создание начальных yml-фаилов
    Warehouse.new
    PzdcMonolith.new
    Shop.new
    OccultLibrary.new
    Options.new # добавим экземпляр опций, чтобы если это первый запуск игры, создать yml-фаил с опциями по умолчанию

    # ход игры
    loop do
      MainRenderer.new(:start_game_screen).display
      choose = gets.strip
      if choose == '0'
        puts "\e[H\e[2J"
        exit
      elsif choose == '2'
        # Лагерь
        CampEngine.new.camp
      elsif choose == '3' # добавим вызов энджина опций
        OptionsEngine.new.main
      else
        # Забег
        load_or_start_new_run()
        Run.new(@hero, @leveling).start if @hero
      end
    end
  end

  # ...

end
```


PZDC_dungeon_2 на Github https://github.com/krillan49/PZDC_dungeon_2


**PZDC_dungeon_2. Добавление новой фичи, на примере опций для выбора скорости анимации. Часть 4**

4. Применим созданный функционал по назначению в энджине `engines\attacks_round`, создадим там метод, который заменит `sleep(1)`

```ruby
class AttacksRound
  def initialize(hero, enemy, messages)

    # ...

    # получаем значение скорости установленное в yml-фаиле
    @enemy_animation_speed = Options.new.get_enemy_actions_animation_speed
  end

  # ...

  private

  # теперь применяем данный метод вместо хардкода sleep(1)
  def sleep_with_enemy_animation_speed
    sleep(@enemy_animation_speed)
  end

end
```


PZDC_dungeon_2 на Github https://github.com/krillan49/PZDC_dungeon_2


**PZDC_dungeon_2. Добавление новой фичи, на примере опций для выбора скорости анимации. Часть 5**

Последовательность экранов


PZDC_dungeon_2 на Github https://github.com/krillan49/PZDC_dungeon_2
