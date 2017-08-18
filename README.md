TangoMan CSV Reader Bundle
==========================

**TangoMan CSV Reader Bundle** provides service for reading csv.


How to install
--------------

With composer

```console
$ composer require tangoman/csv-reader-bundle
```


Enable the bundle
-----------------

Don't forget to enable the bundle in the kernel:

```php
<?php
// app/AppKernel.php

public function registerBundles()
{
    $bundles = array(
        // ...
        new TangoMan\CSVReader\TangoManCSVReader(),
    );
}
```



How to use
----------

Inside your controller:

```php
<?php
namespace AppBundle\Controller;

use AppBundle\Entity\Foobar;
use AppBundle\Form\FileUploadType;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;

/**
 * @Route("/foobars")
 */
class FoobarController extends Controller
{
    /**
     * @Route("/import")
     */
    public function importAction(Request $request)
    {
        $form = $this->createForm(FileUploadType::class);
        $form->handleRequest($request);

        if ($form->isSubmitted()) {

            $file = $request->files->get('file_upload')['file'];

            if (!$file->isValid()) {
                // Upload success check
                $this->get('session')->getFlashBag()->add(
                    'error',
                    'Une erreur s\'est produite lors du transfert.<br/>Veuillez réessayer.'
                );

                return $this->redirectToRoute('app_foobar_import');
            }

            // Security checks
            $validExtensions = ['csv', 'tsv'];
            $clientExtension = $file->getClientOriginalExtension();

            if ($file->getClientMimeType() !== 'application/vnd.ms-excel' &&
                !in_array($clientExtension, $validExtensions)
            ) {

                $this->get('session')->getFlashBag()->add('error', 'Ce format du fichier n\'est pas supporté.');

                return $this->redirectToRoute('app_foobar_import');
            }

            // Get CSV reader service
            $reader = $this->get('services.csv_reader');
            $counter = 0;
            $dupes = 0;
            // File check
            if (is_file($file)) {
                // Init reader service
                $reader->init($file, 0, ';');
                // Load foobar entity
                $em = $this->get('doctrine')->getManager();
                $foobars = $em->getRepository('AppBundle:Foobar');
                // Read current line
                while (false !== ($line = $reader->readLine())) {

                    // Check if foobar with same name exists already
                    $foobar = $foobars->findOneById($line->get('foobar_name'));
                    // When not found persist new foobar
                    if (!$foobar) {
                        $counter++;
                        $foobar = new Foobar();
                        $foobar->setFoobarname($line->get('foobar_name'));

                        // Import string
                        $text = $line->get('foobar_text');
                        if ($text) {
                            $foobar->setBio($text);
                        }

                        // Import array values
                        $list = $line->get('foobar_list');
                        if ($list) {
                            $foobar->setList(explode(',', $line->get('foobar_list')));
                        }

                        // Import DateTime
                        $created = $line->get('foobar_created');
                        if ($created) {
                            $foobar->setCreated(date_create_from_format('Y/m/d H:i:s', $line->get('foobar_created')));
                        }

                        $em->persist($foobar);
                        $em->flush();
                    } else {
                        $dupes++;
                    }
                }
            }

            if ($counter > 0) {
                $msg = $counter.' foobars ont été importés.';
            } else {
                $msg = 'Aucun foobar n\'a été importé.';
            }

            $this->get('session')->getFlashBag()->add('success', $msg);

            return $this->redirectToRoute('app_foobar_index');
        }

        return $this->render(
            'foobar/import.html.twig',
            [
                'form' => $form->createView(),
            ]
        );
    }
}
```


Note
====

If you find any bug please report here : [Issues](https://github.com/TangoMan75/CSVReader/issues/new)

License
=======

Copyrights (c) 2017 Matthias Morin

[![License][license-GPL]][license-url]
Distributed under the GPLv3.0 license.

If you like **TangoMan User Bundle** please star!
And follow me on GitHub: [TangoMan75](https://github.com/TangoMan75)
... And check my other cool projects.

[tangoman.free.fr](http://tangoman.free.fr)

[license-GPL]: https://img.shields.io/badge/Licence-GPLv3.0-green.svg
[license-MIT]: https://img.shields.io/badge/Licence-MIT-green.svg
[license-url]: LICENSE
