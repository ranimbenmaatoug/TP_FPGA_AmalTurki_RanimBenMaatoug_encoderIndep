# TP_FPGA_AmalTurki_RanimBenMaatoug_encoderIndep
Ce repo contient la premiére séance du TP mini_projet Télecran (encoder indépendant)
### Auteurs: Ranim ben maatoug & Amal Turki
### Architecture de l'encodeur
<img width="1536" height="2048" alt="33a9665b-c6dc-473c-bdca-d3caee67f2da" src="https://github.com/user-attachments/assets/1268e20c-fac1-4ed0-8142-0ed7e6a7a62a" />

### Code de l'encodeur en VHDL
```vhdl
library ieee;
use ieee.std_logic_1164.all;
use ieee.numeric_std.all;

entity encoder_indep is
    generic (
        G_WIDTH     : natural := 10;           
        G_CLK_HZ    : natural := 50_000_000;   
        G_SAMPLE_US : natural := 100_000       
    );
    port (
        i_clk   : in  std_logic;
        i_rst_n : in  std_logic;
        i_a     : in  std_logic;
        i_b     : in  std_logic;
        o_leds  : out std_logic_vector(G_WIDTH-1 downto 0)
    );
end entity encoder_indep;

architecture rtl of encoder_indep is

    constant C_DIV : natural := (G_CLK_HZ / 1_000_000) * G_SAMPLE_US;

   
    signal r_div_cnt : natural range 0 to C_DIV-1 := 0;
    signal s_en      : std_logic := '0';

    
    signal r_a_cur, r_a_prev : std_logic := '0';
    signal r_b_cur, r_b_prev : std_logic := '0';

   
    signal s_a_rising, s_a_falling : std_logic;
    signal s_b_rising, s_b_falling : std_logic;

   
    signal s_clockwise, s_cntclockwise : std_logic;

    
    signal r_cnt : unsigned(G_WIDTH-1 downto 0) := (others => '0');
begin

   
    process(i_clk, i_rst_n)
    begin
        if (i_rst_n = '0') then
            r_div_cnt <= 0;
            s_en      <= '0';
        elsif rising_edge(i_clk) then
            if (r_div_cnt = C_DIV-1) then
                r_div_cnt <= 0;
                s_en      <= '1';
            else
                r_div_cnt <= r_div_cnt + 1;
                s_en      <= '0';
            end if;
        end if;
    end process;

   
    process(i_clk, i_rst_n)
    begin
        if (i_rst_n = '0') then
            r_a_cur  <= '0';
            r_a_prev <= '0';
            r_b_cur  <= '0';
            r_b_prev <= '0';
        elsif rising_edge(i_clk) then
            if (s_en = '1') then
                r_a_cur  <= i_a;
                r_a_prev <= r_a_cur;
                r_b_cur  <= i_b;
                r_b_prev <= r_b_cur;
            end if;
        end if;
    end process;

  
    s_a_rising  <= r_a_cur and (not r_a_prev);
    s_a_falling <= (not r_a_cur) and r_a_prev;
    s_b_rising  <= r_b_cur and (not r_b_prev);
    s_b_falling <= (not r_b_cur) and r_b_prev;

   
    s_clockwise    <= (s_a_rising and (not r_b_cur)) or (s_a_falling and r_b_cur);
    s_cntclockwise <= (s_b_rising and (not r_a_cur)) or (s_b_falling and r_a_cur);

    
    process(i_clk, i_rst_n)
    begin
        if (i_rst_n = '0') then
            r_cnt <= (others => '0');
        elsif rising_edge(i_clk) then
            if (s_en = '1') then
                if (s_clockwise = '1') then
                    r_cnt <= r_cnt + 1;
                elsif (s_cntclockwise = '1') then
                    r_cnt <= r_cnt - 1;
                end if;
            end if;
        end if;
    end process;

    o_leds <= std_logic_vector(r_cnt);

end architecture rtl;
```
#### Explication du code

- **Generics** : `G_WIDTH` rend la taille du compteur configurable (10 pour les LED, réutilisable plus tard pour les coordonnées de l'écran). `G_CLK_HZ` et `G_SAMPLE_US` permettent de calculer le diviseur `C_DIV` à partir de la fréquence d'horloge et de la période voulue (100 ms sur la carte, 1 µs en simulation), sans toucher au code.

- **Process 1 (diviseur)** : génère `s_en`, une impulsion d'un seul cycle toutes les 100 ms. C'est un *clock enable*, pas une nouvelle horloge : tout le design reste cadencé par `i_clk`, ce qui évite de créer une horloge dérivée dans la logique.

- **Process 2 (bascules D)** : les signaux n'étant mis à jour qu'en fin de process, `r_a_prev` reçoit l'ancienne valeur de `r_a_cur`. On dispose donc de la valeur actuelle et de la valeur précédente de chaque voie. L'échantillonnage toutes les 100 ms élimine les rebonds mécaniques, et la première bascule synchronise les entrées, qui sont asynchrones par rapport à `i_clk`.

- **Détection de fronts** : logique combinatoire `cur AND NOT prev` pour un front montant, `NOT cur AND prev` pour un front descendant. Comme `cur` et `prev` ne changent qu'à chaque `s_en`, un front reste visible pendant 100 ms : c'est pourquoi le compteur n'agit lui aussi que lorsque `s_en = '1'`, pour ne compter chaque front qu'une seule fois.

- **Décision du sens** : `s_clockwise` traduit les deux conditions d'incrémentation du sujet (front sur A combiné à l'état de B), et `s_cntclockwise` les deux conditions de décrémentation (front sur B combiné à l'état de A). L'état de l'autre voie est lu sur sa version synchronisée (`r_x_cur`).

- **Process 3 (compteur)** : +1 sur `s_clockwise`, −1 sur `s_cntclockwise`, valeur conservée sinon. Le registre `unsigned` boucle modulo 2^G_WIDTH (après 1023 vient 0).
### RTL Viewer
<img width="1860" height="536" alt="rtlViewer" src="https://github.com/user-attachments/assets/a2e77867-cd89-4790-8dab-e96af2b79583" />

### Simulation MedelSim
 Pour vérifier le composant sans la carte, nous avons écrit un testbench.

 **testBench**
 ```vhdl

library ieee;
use ieee.std_logic_1164.all;
use ieee.numeric_std.all;

entity tb_encoder_indep is
end entity tb_encoder_indep;

architecture sim of tb_encoder_indep is
    constant C_PERIOD : time := 20 ns;     
    constant C_STEP   : time := 5 us;     

    signal s_clk   : std_logic := '0';
    signal s_rst_n : std_logic := '0';
    signal s_a     : std_logic := '0';
    signal s_b     : std_logic := '0';
    signal s_leds  : std_logic_vector(9 downto 0);
    signal s_end   : boolean := false;
begin

    dut : entity work.encoder_indep
        generic map (G_WIDTH => 10, G_CLK_HZ => 50_000_000, G_SAMPLE_US => 1)
        port map (
            i_clk   => s_clk,
            i_rst_n => s_rst_n,
            i_a     => s_a,
            i_b     => s_b,
            o_leds  => s_leds
        );

    s_clk <= not s_clk after C_PERIOD/2 when not s_end else '0';

    process
        procedure tourner_droite is   
        begin
            s_a <= '1'; wait for C_STEP;
            s_b <= '1'; wait for C_STEP;
            s_a <= '0'; wait for C_STEP;
            s_b <= '0'; wait for C_STEP;
        end procedure;

        procedure tourner_gauche is  
        begin
            s_b <= '1'; wait for C_STEP;
            s_a <= '1'; wait for C_STEP;
            s_b <= '0'; wait for C_STEP;
            s_a <= '0'; wait for C_STEP;
        end procedure;
    begin
        s_rst_n <= '0'; wait for 5*C_PERIOD;
        s_rst_n <= '1'; wait for C_STEP;
        assert unsigned(s_leds) = 0 report "Erreur : apres reset" severity error;

        for i in 1 to 3 loop tourner_droite; end loop;
        assert unsigned(s_leds) = 6 report "Erreur : droite, attendu 6" severity error;

        tourner_gauche;
        assert unsigned(s_leds) = 4 report "Erreur : gauche, attendu 4" severity error;

        for i in 1 to 3 loop tourner_gauche; end loop;
        assert unsigned(s_leds) = 1022 report "Erreur : attendu 1022" severity error;

        report "Fin de simulation : tous les tests sont passes" severity note;
        s_end <= true;
        wait;
    end process;

end architecture sim;
```

**Resultat de la simulation sur ModelSim**
<img width="1913" height="1136" alt="e66788d7-7877-47f4-bc35-525f744eaee2 (1)" src="https://github.com/user-attachments/assets/2460186c-a3a2-4d56-adf2-782b4a37add6" />

### Test sur la carte pour 100ms

https://github.com/user-attachments/assets/9f92a59b-7159-417c-b331-a6a8e0e8e051

Au début, on clique sur Key0 ce qui fait que tous les LEDS sont éteints puis on fait tourner la molette gauche  d'un cran dans le sens horaire ce qui fait allumer le premier LED puis un cran de plus dans le même sens ce qui allume un LED de plus (allant des LEDS ayant les poids les plus faibles au plus forts)
Quand on change de sens de rotation de la molette, les combinaisons reviennent en arrière : la valeur redescend.

### Remarque
- **Réactivité et période d'échantillonnage** : avec une période de 100 ms, l'encodeur n'est lu que 10 fois par seconde. En tournant lentement, chaque cran est bien compté, mais dès qu'on tourne plus vite, des pas sont perdus : plusieurs changements de A et B se produisent entre deux lectures et ne sont pas vus. Si A et B changent tous les deux entre deux lectures, le circuit voit deux fronts en même temps et ne sait plus lequel est arrivé en premier, ce qui peut même faire compter dans le mauvais sens. En réduisant la période (par exemple à `G_SAMPLE_US => 1_000`, soit 1 ms), l'encodeur devient nettement plus réactif : les rotations rapides sont suivies correctement.




