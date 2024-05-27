
const express = require('express');
const multer = require('multer');
const { Pool } = require('pg');
const bodyParser = require('body-parser');
const path = require('path');

const app = express();
const port = 3000;

const pool = new Pool({
    user: 'yourusername',
    host: 'localhost',
    database: 'filetracking',
    password: 'yourpassword',
    port: 5432
});

app.use(bodyParser.json());
app.use(bodyParser.urlencoded({ extended: true }));
app.use(express.static('public'));

const upload = multer({ dest: 'uploads/' });

app.post('/upload', upload.single('file'), async (req, res) => {
    const file = req.file;
    const { fromOffice, toOffice, actorId } = req.body;
    try {
        const result = await pool.query(
            'INSERT INTO files (name, path, uploader_id) VALUES ($1, $2, $3) RETURNING id',
            [file.originalname, file.path, actorId]
        );
        const fileId = result.rows[0].id;
        await pool.query(
            'INSERT INTO file_movements (file_id, from_office_id, to_office_id, status, actor_id) VALUES ($1, $2, $3, $4, $5)',
            [fileId, fromOffice, toOffice, 'initiated', actorId]
        );
        res.status(200).send('File uploaded and transfer initiated');
    } catch (err) {
        console.error(err);
        res.status(500).send('File upload failed');
    }
});

app.get('/track/:fileId', async (req, res) => {
    const fileId = req.params.fileId;
    try {
        const trackingResult = await pool.query('SELECT * FROM file_movements WHERE file_id = $1', [fileId]);
        res.status(200).json(trackingResult.rows);
    } catch (err) {
        console.error(err);
        res.status(500).send('Failed to retrieve tracking information');
    }
});

app.listen(port, () => {
    console.log(`File tracking app listening at http://localhost:${port}`);
});
