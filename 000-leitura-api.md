const jsonFilePath = __dirname + '/data.temp.json'; // Define o caminho do arquivo de dados.
const list: string[] = await loadFromFile(); // Carrega a lista inicial do arquivo.

async function loadFromFile() { // Inicia função de leitura de arquivo.
  try { // Tenta executar o bloco de leitura.
    const file = Bun.file(jsonFilePath); // Faz a referência ao arquivo via Bun.
    const content = await file.text(); // Extrai o conteúdo do arquivo como texto.
    return JSON.parse(content) as string[]; // Converte o texto JSON para array.
  } catch (error: any) { // Captura erros caso a leitura falhe.
    if (error.code === 'ENOENT') // Verifica se o erro é arquivo inexistente.
      return []; // Retorna array vazio se não houver arquivo.
    throw error; // Lança outros tipos de erro.
  }
}

async function saveToFile() { // Inicia função de escrita no arquivo.
  try { // Tenta executar a persistência de dados.
    await Bun.write(jsonFilePath, JSON.stringify(list)); // Salva a lista convertida em JSON.
  } catch (error: any) { // Captura erros durante a gravação.
   throw new Error("Erro ao salvar os dados no arquivo: " + error.message); // Exibe mensagem de erro customizada.
  }
}

async function addItem(item: string) { // Inicia função de adição de itens.
  list.push(item); // Insere o novo item no array local.
  await saveToFile(); // Sincroniza a alteração com o arquivo.
}

async function getItems() { // Inicia função de retorno da lista.
  return list; // Retorna o array de strings atual.
}

async function updateItem(index: number, newItem: string) { // Inicia função de edição por índice.
  if (index < 0 || index >= list.length) // Valida se o índice existe na lista.
    throw new Error("Index fora dos limites"); // Erro se o índice for inválido.
  list[index] = newItem; // Substitui o valor na posição informada.
  await saveToFile(); // Sincroniza a edição com o arquivo.
}

async function removeItem(index: number) { // Inicia função de exclusão por índice.
  if (index < 0 || index >= list.length) // Valida se o índice existe na lista.
    throw new Error("Index fora dos limites"); // Erro se o índice for inválido.
  list.splice(index, 1); // Remove 1 elemento da posição informada.
  await saveToFile(); // Sincroniza a remoção com o arquivo.
}

export default { addItem, getItems, updateItem, removeItem }; // Exporta métodos para uso externo.

--------------------------------------------------------------------------------------------------------------------------------

import todo from "./core.ts"; // Importa a lógica de negócio do core.

const server = Bun.serve({ // Inicia a instância do servidor Bun.
  port: 3000, // Define a porta de escuta do servidor.

  routes: { // Define o objeto de roteamento da API.
    "/api/todo": { // Rota base para operações coletivas.
      GET: async () => { // Define o método de busca.
        const items = await todo.getItems(); // Busca todos os itens da lista.
        return Response.json(items); // Retorna a lista em formato JSON.
      },
     POST: async (req) => { // Define o método de criação.
        const data = await req.json() as any; // Extrai o corpo da requisição.
        const item = data.item || null; // Obtém o campo 'item' do JSON.
        if (!item) // Valida se o item foi enviado.
          return Response.json('Por favor, forneça um item para adicionar.', { status: 400 }); // Erro de falta de dado.
        await todo.addItem(item); // Chama a função para salvar o item.
        return Response.json(data); // Retorna o dado criado com sucesso.
      },
    },
    "/api/todo/:index": { // Rota para operações em itens específicos.
      PUT: async (req) => { // Define o método de atualização.
        const index = parseInt(req.params.index); // Converte o parâmetro da URL em número.
        if (isNaN(index)) // Valida se o índice é numérico.
          return Response.json('Índice inválido. um número inteiro é esperado.', { status: 400 }); // Erro de tipo de índice.
        const data = await req.json() as any; // Extrai o corpo da requisição.
        const newItem = data.newItem || null; // Obtém o novo valor do item.
        if (!newItem) // Valida se o novo valor foi enviado.
          return Response.json('Por favor, forneça um novo item para atualizar.', { status: 400 }); // Erro de dado ausente.
        try { // Tenta atualizar o item no core.
          await todo.updateItem(index, newItem); // Executa a lógica de atualização.
          return Response.json(`Item no índice ${index} atualizado para "${newItem}".`); // Sucesso na atualização.
        } catch (error: any) { // Captura erros da lógica do core.
          return Response.json(error.message, { status: 400 }); // Retorna a mensagem de erro.
        }
      },
      DELETE: async (req) => { // Define o método de exclusão.
        const index = parseInt(req.params.index); // Converte o parâmetro da URL em número.
        if (isNaN(index)) // Valida se o índice é numérico.
          return Response.json('Índice inválido.', { status: 400 }); // Erro de índice inválido.
        try { // Tenta remover o item no core.
          await todo.removeItem(index); // Executa a lógica de remoção.
          return Response.json(`Item no índice ${index} removido com sucesso.`); // Sucesso na remoção.
        } catch (error: any) { // Captura erros da lógica do core.
          return Response.json(error.message, { status: 400 }); // Retorna a mensagem de erro.
        }
      },
    },
  },

  async fetch(req) { // Intercepta requisições não tratadas nas rotas.
    const url = new URL(req.url); // Analisa a URL da requisição.
    const path = url.pathname; // Extrai o caminho da URL.
    const filePath = (path === '/') // Verifica se o caminho é a raiz.
      ? './public/index.html' // Define o arquivo index como padrão.
      : `./public${path}`; // Mapeia o caminho para a pasta public.
    const file = Bun.file(filePath); // Tenta carregar o arquivo estático.
    if (await file.exists()) // Verifica se o arquivo físico existe.
      return new Response(file); // Serve o arquivo estático encontrado.
    return new Response(`Not Found`, { status: 404 }); // Retorna 404 se nada for encontrado.
  },
});

console.log(`Server running at http://localhost:${server.port}`); // Loga o status de execução do servidor.
