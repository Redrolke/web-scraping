import time
import pandas as pd
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.common.by import By
from webdriver_manager.chrome import ChromeDriverManager

def scrape_x_profile(username, max_scrolls=3):
    # Configurando o navegador Chrome
    options = webdriver.ChromeOptions()
    # options.add_argument('--headless') # Descomente se não quiser ver a janela (não recomendado no X devido ao login)
    driver = webdriver.Chrome(service=Service(ChromeDriverManager().install()), options=options)

    url = f"https://x.com/{username}"
    driver.get(url)

    print("IMPORTANTE: Você tem 15 segundos para fazer login manualmente na janela do Chrome, se o X pedir!")
    time.sleep(15) # Tempo para você logar ou para a página carregar completamente

    dados_extraidos = []
    tweets_processados = set() # Para evitar duplicatas

    print(f"Iniciando a coleta de dados do usuário: @{username}...")

    # Rolar a página para carregar mais tweets
    for scroll in range(max_scrolls):
        # Encontrar todos os elementos de tweet (o X usa a tag <article> para cada post)
        tweets = driver.find_elements(By.XPATH, '//article[@data-testid="tweet"]')
        
        for tweet in tweets:
            try:
                # Extraindo o texto do post
                texto_element = tweet.find_element(By.XPATH, './/div[@data-testid="tweetText"]')
                texto = texto_element.text
            except:
                texto = "Sem texto/Apenas mídia"

            try:
                # Extraindo a data/hora (A tag <time> guarda a data em formato ISO no atributo datetime)
                data_element = tweet.find_element(By.XPATH, './/time')
                data_postagem = data_element.get_attribute('datetime')
            except:
                data_postagem = "Data não encontrada"

            try:
                # Extraindo o autor (Pega o texto principal de identificação)
                autor_element = tweet.find_element(By.XPATH, './/div[@data-testid="User-Name"]')
                autor = autor_element.text.split('\n')[0] # Pega apenas o Nome de Exibição
            except:
                autor = username

            # Criar um identificador único para o post (usando data e primeiros caracteres do texto)
            tweet_id = f"{data_postagem}-{texto[:20]}"

            if tweet_id not in tweets_processados and texto != "Sem texto/Apenas mídia":
                tweets_processados.add(tweet_id)
                dados_extraidos.append({
                    "Autor": autor,
                    "Username": f"@{username}",
                    "Data": data_postagem,
                    "Texto_do_Post": texto
                })

        # Rolar para baixo para carregar mais posts
        driver.execute_script("window.scrollTo(0, document.body.scrollHeight);")
        time.sleep(3) # Pausa para o X carregar os novos elementos via AJAX

    driver.quit()
    return dados_extraidos

# --- Execução do Script ---
if __name__ == "__main__":
    usuario_alvo = "elonmusk" # Troque pelo usuário que deseja coletar
    
    # Executa a função (aumente o max_scrolls se quiser coletar mais posts)
    dados = scrape_x_profile(usuario_alvo, max_scrolls=5)

    # Verifica se encontrou dados e salva em CSV
    if dados:
        df = pd.DataFrame(dados)
        nome_arquivo = f"tweets_{usuario_alvo}.csv"
        df.to_csv(nome_arquivo, index=False, encoding='utf-8')
        print(f"\nColeta concluída com sucesso! Foram extraídos {len(dados)} posts.")
        print(f"Os dados foram salvos no arquivo: {nome_arquivo}")
    else:
        print("\nNenhum dado foi coletado. Verifique se o X bloqueou o acesso ou exigiu login.")
