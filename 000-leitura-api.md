const jsonFilePath = __dirname + '/data.temp.json'; // caminho do arquivo data.temp.json (que armazena os dados).
const list: string[] = await loadFromFile(); // array de strings carregadas do arquivo JSON.

// Função para carregar o conteúdo do arquivo JSON e retornar como string.
async function loadFromFile() {
  try {
    const file = Bun.file(jsonFilePath);
    const content = await file.text();
    return JSON.parse(content) as string[];
  } catch (error: any) {
    if (error.code === 'ENOENT')
      return [];
    throw error;
  }
}

// Função para salvar itens na lista.
async function saveToFile() {
  try {
    await Bun.write(jsonFilePath, JSON.stringify(list));
  } catch (error: any) {
   throw new Error("Erro ao salvar os dados no arquivo: " + error.message);
  }
}

// Função para adicionar itens à lista.
async function addItem(item: string) {
  list.push(item);
  await saveToFile();
}

// Função para listar os itens da lista.
async function getItems() {
  return list;
}

// Função que atualiza (através do index) itens já existentes da lista.
async function updateItem(index: number, newItem: string) {
  if (index < 0 || index >= list.length)
    throw new Error("Index fora dos limites");
  list[index] = newItem;
  await saveToFile();
}

// Função que remove itens (através do index) da lista.
async function removeItem(index: number) {
  if (index < 0 || index >= list.length)
    throw new Error("Index fora dos limites");
  list.splice(index, 1);
  await saveToFile();
}

// Exporta todas as funções deste arquivo core.ts, para serem reutilizadas em outros arquivos, nesse caso no server.ts.
export default { addItem, getItems, updateItem, removeItem };

-----------------------------------------------------------------------------------------------------------

// Importa as funções que o arquivo core.ts faz.
import todo from "./core.ts"; 

// Cria um servidor bun na porta 3000.
const server = Bun.serve({
  port: 3000,

// Cria as possíveis rotas da url.
  routes: {

// Rota que permite realizar ações da TodoList que não necesitam a informação do index do item na url.
    "/api/todo": {

// Método que lista os itens da lista.
      GET: async () => {
        const items = await todo.getItems()
        return Response.json(items)
      },

// Método que adiciona novos itens à lista.
      POST: async (req) => {
        const data = await req.json() as any;
        const item = data.item || null;
        if (!item)
          return Response.json('Por favor, forneça um item para adicionar.', { status: 400 });
        await todo.addItem(item);
        return Response.json(data);
      },
    },

// Rota que permite realizar ações da TodoList que necessitam a informação do index na url.
    "/api/todo/:index": {

// Método que altera itens da lista.
      PUT: async (req) => {
        const index = parseInt(req.params.index);
        if (isNaN(index))
          return Response.json('Índice inválido. um número inteiro é esperado.', { status: 400 });
        const data = await req.json() as any;
        const newItem = data.newItem || null;
        if (!newItem)
          return Response.json('Por favor, forneça um novo item para atualizar.', { status: 400 });
        try {
          await todo.updateItem(index, newItem);
          return Response.json(`Item no índice ${index} atualizado para "${newItem}".`);
        } catch (error: any) {
          return Response.json(error.message, { status: 400 });
        }
      },

// Método que deleta itens da lista.
      DELETE: async (req) => {
        const index = parseInt(req.params.index);
        if (isNaN(index))
          return Response.json('Índice inválido.', { status: 400 });
        try {
          await todo.removeItem(index);
          return Response.json(`Item no índice ${index} removido com sucesso.`);
        } catch (error: any) {
          return Response.json(error.message, { status: 400 });
        }
      },
    },
  },

// Faz com que não seja necessário adicionar rota por rota no código, em vez disso cria uma função para procurar (depois do /public) os arquivos da pasta src que tem o mesmo nome do que seria buscado na url. 
  async fetch(req) {
    // http://localhost:3000/api/todo/1&chave=valor
    const url = new URL(req.url);
    const path = url.pathname;
    const filePath = (path === '/') // Faz com que se não houver um caminho após a /, seja devolvido o index.html
      ? './public/index.html'
      : `./public${path}`;
    const file = Bun.file(filePath);
    if (await file.exists()) 
      return new Response(file);
    return new Response(`Not Found`, { status: 404 });
  },
});

console.log(`Server running at http://localhost:${server.port}`);
