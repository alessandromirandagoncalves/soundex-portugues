# soundex-portugues
Similar à função Soundex do SQL Server mas levando em consideração a língua portuguesa

Trabalho com bancos de dados há muitos anos (principalmente programação - procedures, views -, performance com criação de índices e análise de queries e ETL - migração de dados).
Algo que me aconteceu foi que que a empresa queria unificar cidades iguais mas que houve erro de digitação e constavam como cidade diferentes. Uma análise manual seria custosa e sujeita a erros.
A função Soundex não atendia para descobrir cidades iguais mas com pequenos erros de digitação.

Para aqueles que precisam descobrir similaridade fonética entre palavras, existe uma função chamada Soundex no SQL Server que converte uma cadeia de caracteres alfanumérica em um código de quatro caracteres baseado em como a cadeia de caracteres soa quando falada em inglês. O primeiro caractere do código é o primeiro caractere de character_expression, convertido em maiúsculas. O segundo até o quarto caractere do código são números que representam as letras da expressão. As letras A, E, I, O, U, H, We Y são ignoradas, a menos que sejam a primeira letra da cadeia de caracteres. Zeros serão adicionados ao término, se necessário, para gerar um código de quatro caracteres

<img width="866" height="239" alt="image" src="https://github.com/user-attachments/assets/c9602362-4a00-40a1-bbf4-117e51a0dbf4" />

A pronúncia de Smith é muito próxima de Smythe e o resultado indica que foneticamente são equivalentes (retorna S530 para cada uma da palavra passada)

Então resolvi criar uma função chamada fn_similar onde passo o nome das duas cidades e o máximo de diferença de letras e a análise é feita considerando-se a língua portuguesa (s, ss e ç com sons parecidos dentre outros)
Se tivermos duas cidades no banco ("Crato" e "Cratos") que deveriam ser a mesma (o "s" foi um erro), o Soundex retorna diferente, como observado abaixo:

<img width="448" height="156" alt="image" src="https://github.com/user-attachments/assets/e71891e8-5432-44fa-ac03-56399638b3ce" />



Mas utilizando a função fn_similar ela trará o resultado como "1 - similar", como visto a seguir.
<img width="435" height="146" alt="image" src="https://github.com/user-attachments/assets/ae4cfb2f-fc67-41d2-8af4-6fffeec6681f" />

No exemplo acima a "distância fonética" máxima é de 1, ou seja, se houver até uma letra de diferença ela retornar 1 (similar). Se quiser ser mais flexível poderia colocar 2 na chamada da função, que iria considerar similar até 2 letras de diferença.
Ex: fn_similar('Crato','Cratoos,2) retornaria 1, mesmo com 2 letras de diferença entre elas.
A função fn_similar usa  função fn_soundex_pt (a soundex adaptada para o português - também descrita no código)
A partir daí pode-se analisar com mais profundidade e descobrir se realmente as cidades seriam as mesmas e proceder às alterações, se for o caso.
Desta forma, muita coisa é filtrada e a análise visual é reduzida para poucos casos.

O código é descrito a seguir

```sql

CREATE FUNCTION fn_Soundex_PT (@word VARCHAR(250))
RETURNS VARCHAR(16)
AS
BEGIN
    DECLARE @encoded NVARCHAR(100)
    SET @word = UPPER(@word)
    
    -- Substituições específicas do português
    SET @word = REPLACE(@word, 'Á', 'A')
    SET @word = REPLACE(@word, 'Ã', 'AN')
    SET @word = REPLACE(@word, 'Â', 'A')
    SET @word = REPLACE(@word, 'À', 'A')
    SET @word = REPLACE(@word, 'Í', 'I')
    SET @word = REPLACE(@word, 'Ì', 'I')
    SET @word = REPLACE(@word, 'É', 'E')
    SET @word = REPLACE(@word, 'È', 'E')
    SET @word = REPLACE(@word, 'Ê', 'E')
    SET @word = REPLACE(@word, 'Ó', 'O')
    SET @word = REPLACE(@word, 'Õ', 'O')
    SET @word = REPLACE(@word, 'Ô', 'O')
    SET @word = REPLACE(@word, 'Ú', 'U')
    SET @word = REPLACE(@word, 'Û', 'U')
    SET @word = REPLACE(@word, 'Ç', 'S')
    SET @word = REPLACE(@word, 'Ñ', 'N')
    SET @word = REPLACE(@word, 'NN', 'N')
    SET @word = REPLACE(@word, 'NH', 'N')
    SET @word = REPLACE(@word, 'LH', 'L')
    SET @word = REPLACE(@word, 'CH', 'X')
    SET @word = REPLACE(@word, 'SH', 'X')
    SET @word = REPLACE(@word, 'Q', 'K')
    SET @word = REPLACE(@word, 'C', 'K')
    SET @word = REPLACE(@word, 'Z', 'S')
    SET @word = REPLACE(@word, 'PH', 'F')
    SET @word = REPLACE(@word, 'G', 'J')
    SET @word = REPLACE(@word, 'Y', 'I')
    SET @word = REPLACE(@word, ' DE ', '')
    SET @word = REPLACE(@word, ' DA ', '')
    SET @word = REPLACE(@word, ' DAS ', '')
    SET @word = REPLACE(@word, ' DO ', '')
    SET @word = REPLACE(@word, ' DOS ', '')
    
    DECLARE @first_letter CHAR(1) = LEFT(@word, 1)
    DECLARE @i BIGINT = 2
    DECLARE @char CHAR(1)
    DECLARE @last_digit CHAR(1) = ''
    SET @encoded = @first_letter
    
    WHILE @i <= LEN(@word) AND LEN(@encoded) < 16
    BEGIN
        SET @char = SUBSTRING(@word, @i, 1)
        
        DECLARE @digit CHAR(1)
        SET @digit = CASE
            WHEN @char IN ('B', 'F', 'P', 'V') THEN '1'
            WHEN @char IN ('C', 'G', 'J', 'K', 'Q', 'S', 'X', 'Z') THEN '2'
            WHEN @char IN ('D', 'T') THEN '3'
            WHEN @char = 'L' THEN '4'
            WHEN @char IN ('M', 'N') THEN '5'
            WHEN @char = 'R' THEN '6'
            WHEN @char = 'A' THEN '7'
            WHEN @char = 'E' THEN '8'
            WHEN @char IN ('O','U') THEN '9'
            ELSE ''
        END
        
        IF @digit <> '' AND @digit <> @last_digit
        BEGIN
            SET @encoded = @encoded + @digit
            SET @last_digit = @digit
        END
        
        SET @i = @i + 1
    END
    
    SET @encoded = LEFT(@encoded, 16)
    
    RETURN @encoded
END

-- Usa a fn_Soundex_PT para comparar
CREATE FUNCTION fn_Similar (@word1 VARCHAR(250),@word2 VARCHAR(250), @diferenca float = 2)
RETURNS bit
AS
-- Comparar @word1 com @word2
-- A @diferenca leva em consideração a sensibilidade de correspondência. Quanto menor, mais próximas devem ser para retornar 1 (True)
-- Ex: @diferença = 2 significa que até duas letras de diferença após aplicar fn_soundex_pt (faltando/sobrando ou mesmo diferentes retorna 1)
-- Caso contrário retorna 0. Nos testes efetuados @diferenca = 2 trouxe ótimos resultados
BEGIN
	DECLARE @Result bit = 0
	DECLARE @palavra1Cripto varchar(250)
	DECLARE @palavra2Cripto varchar(250)
	DECLARE @Distancia bigint = 0
	DECLARE @i int

    --Substitui letras com sons parecidos. Ex: á e à por a...
	Select @palavra1Cripto = dbo.fn_soundex_pt(@word1)
	Select @palavra2Cripto = dbo.fn_soundex_pt(@word2)

	--Iguala o tamanho das variáveis, prenchendo com 0 se extrapolar
	if len(@palavra1Cripto) < len(@palavra2Cripto) set @palavra1Cripto = @palavra1Cripto + replicate('0',len(@palavra2Cripto)-len(@palavra1Cripto))
	if len(@palavra1Cripto) > len(@palavra2Cripto) set @palavra2Cripto = @palavra2Cripto + replicate('0',len(@palavra1Cripto)-len(@palavra2Cripto))
	SET @i = 2
	WHILE @i <= len (@palavra1Cripto)
	BEGIN
		SET @Distancia = @Distancia + abs(cast(SUBSTRING(@palavra1Cripto,@i,1) as integer)-cast(SUBSTRING(@palavra2Cripto,@i,1) as integer))
		SET @i = @i + 1
	END
	if left(@palavra1Cripto,1) <> left(@palavra2Cripto,1) set @Distancia=@Distancia+5

	if @Distancia/cast(LEN(@palavra1Cripto) as float) <= @diferenca SET @RESULT = 1
	RETURN @Result

END

-- Para testar:
Select dbo.fn_similar('crato','cratos',1)
```
dúvidas:
linkedin: [Alessandro Miranda](https://www.linkedin.com/in/alessandromirandagoncalves/)

