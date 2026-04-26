[!DOCTYPE html.html](https://github.com/user-attachments/files/27099565/DOCTYPE.html.html)
<!DOCTYPE html>
<html lang="bg">
<head>
    <meta charset="UTF-8">
    <title>My Cloud</title>
    <style>
        body {
            font-family: Arial;
            background: #0f172a;
            color: white;
            text-align: center;
            padding: 40px;
        }

        .container {
            background: #1e293b;
            padding: 20px;
            border-radius: 15px;
            max-width: 500px;
            margin: auto;
        }

        input {
            margin: 10px;
        }

        .file {
            background: #334155;
            margin: 5px;
            padding: 10px;
            border-radius: 10px;
        }

        button {
            padding: 8px 15px;
            border: none;
            border-radius: 10px;
            background: #38bdf8;
            cursor: pointer;
        }
    </style>
</head>
<body>

    <div class="container">
        <h1>☁️ My Cloud</h1>

        <input type="file" id="fileInput">
        <br>
        <button onclick="uploadFile()">Качи файл</button>

        <h3>Файлове:</h3>
        <div id="fileList"></div>
    </div>

<script>
let files = JSON.parse(localStorage.getItem("files")) || [];

function renderFiles() {
    const list = document.getElementById("fileList");
    list.innerHTML = "";

    files.forEach((file, index) => {
        const div = document.createElement("div");
        div.className = "file";
        div.innerHTML = `
            ${file.name}
            <br>
            <a href="${file.data}" download="${file.name}">
                <button>Изтегли</button>
            </a>
            <button onclick="deleteFile(${index})">Изтрий</button>
        `;
        list.appendChild(div);
    });
}

function uploadFile() {
    const input = document.getElementById("fileInput");
    const file = input.files[0];

    if (!file) return;

    const reader = new FileReader();
    reader.onload = function(e) {
        files.push({
            name: file.name,
            data: e.target.result
        });

        localStorage.setItem("files", JSON.stringify(files));
        renderFiles();
    };

    reader.readAsDataURL(file);
}

function deleteFile(index) {
    files.splice(index, 1);
    localStorage.setItem("files", JSON.stringify(files));
    renderFiles();
}

renderFiles();
</script>

</body>
</html>
